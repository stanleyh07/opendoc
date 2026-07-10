## 🧩 **OpenWrt Container Router — 完整自動化建置手冊

本手冊提供一套完整、自動化、可重現的架構，使：

- Host **不使用任何實體 NIC**
- 所有實體 NIC（eth1~eth8）完全由 OpenWrt container 控制
- Host 透過 veth → OpenWrt → WAN 上網
- Host 完全受 OpenWrt firewall/NAT 保護
- 所有設定在開機後自動完成（無需手動操作）

---

# # 1. 系統架構

```
                ┌────────────────────────────────┐
                │            Host OS             │
                │                                │
                │   veth-host (10.0.0.2/24)      │
                │        ↓ default route         │
                └──────────────┬─────────────────┘
                               │
                               ▼
                ┌────────────────────────────────┐
                │       OpenWrt Container        │
                │                                │
                │  veth-openwrt (10.0.0.1/24)    │
                │  eth1~eth7 (LAN 實體 NIC)      │
                │  eth8 (WAN 實體 NIC)           │
                │  NAT + Firewall                │
                └────────────────────────────────┘
```

---

# # 2. Host Kernel 設定（CONFIG）

以下為建議啟用的 kernel config：

```
CONFIG_NAMESPACES=y
CONFIG_NET_NS=y
CONFIG_PID_NS=y
CONFIG_IPC_NS=y
CONFIG_UTS_NS=y
CONFIG_USER_NS=y
CONFIG_CGROUPS=y
CONFIG_CGROUP_NET_PRIO=y
CONFIG_CGROUP_NET_CLASSID=y
CONFIG_VETH=y
CONFIG_BRIDGE=y
CONFIG_BRIDGE_NETFILTER=y
CONFIG_NETFILTER=y
CONFIG_NETFILTER_ADVANCED=y
CONFIG_VLAN_8021Q=y
CONFIG_NETFILTER_XT_MATCH_VLAN=y
CONFIG_PPP=y
CONFIG_PPPOE=y
CONFIG_PPP_ASYNC=y
CONFIG_PPP_SYNC_TTY=y
CONFIG_NETFILTER_XT_TARGET_MASQUERADE=y
CONFIG_IP_NF_NAT=y
CONFIG_NF_NAT=y
CONFIG_NF_CONNTRACK=y
CONFIG_NF_CONNTRACK_IPV4=y
CONFIG_NF_CONNTRACK_IPV6=y
CONFIG_TUN=y
```

NIC 驅動依硬體選擇，例如：

Intel：

```
CONFIG_E1000E=y
CONFIG_IGB=y
CONFIG_IXGBE=y
```

Realtek：

```
CONFIG_R8169=y
```

---

# # 3. Host RootFS 設定

停用所有會自動管理 NIC 的服務：

```
systemctl disable --now NetworkManager || true
systemctl disable --now systemd-networkd || true
```

確保 eth1~eth8 沒有 IP：

```
ip addr show eth1
ip addr show eth8
```

---

# # 4. 啟動 OpenWrt Container（自動化前置）

```
docker run -d \
  --name openwrt \
  --network none \
  --cap-add=NET_ADMIN \
  --cap-add=NET_RAW \
  --restart=unless-stopped \
  openwrt:latest
```

---

# # 5. 開機自動化 Script（核心）

建立：

```
nano /usr/local/sbin/openwrt-net-setup.sh
```

內容如下（完整可用版）：

```bash
#!/usr/bin/env bash
set -e

CONTAINER_NAME="openwrt"
HOST_VETH="veth-host"
GUEST_VETH="veth-openwrt"
HOST_IP="10.0.0.2/24"
GUEST_IP="10.0.0.1"
WAN_IF="eth8"
LAN_IFS="eth1 eth2 eth3 eth4 eth5 eth6 eth7"

wait_for_container() {
    for i in {1..20}; do
        if docker inspect "$CONTAINER_NAME" >/dev/null 2>&1; then
            break
        fi
        sleep 1
    done
}

get_container_pid() {
    docker inspect -f '{{.State.Pid}}' "$CONTAINER_NAME"
}

move_interfaces_to_container() {
    local PID="$1"
    for IF in $LAN_IFS $WAN_IF; do
        if ip link show "$IF" >/dev/null 2>&1; then
            ip link set "$IF" down || true
            ip addr flush dev "$IF" || true
            ip link set "$IF" netns "$PID"
        fi
    done
}

setup_veth_pair() {
    local PID="$1"

    if ! ip link show "$HOST_VETH" >/dev/null 2>&1; then
        ip link add "$HOST_VETH" type veth peer name "$GUEST_VETH"
    fi

    ip link set "$HOST_VETH" up
    ip addr flush dev "$HOST_VETH" || true
    ip addr add "$HOST_IP" dev "$HOST_VETH"

    ip link set "$GUEST_VETH" netns "$PID"

    ip route replace default via "$GUEST_IP" dev "$HOST_VETH"
}

docker start "$CONTAINER_NAME"
wait_for_container
PID=$(get_container_pid)
move_interfaces_to_container "$PID"
setup_veth_pair "$PID"
exit 0
```

設定執行權限：

```
chmod +x /usr/local/sbin/openwrt-net-setup.sh
```

---

# # 6. systemd 自動化

建立：

```
nano /etc/systemd/system/openwrt-net.service
```

內容：

```ini
[Unit]
Description=Setup OpenWrt network namespace and NICs
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/openwrt-net-setup.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

啟用：

```
systemctl daemon-reload
systemctl enable openwrt-net
systemctl start openwrt-net
```

---

# # 7. OpenWrt 設定（自動化後仍需一次性設定）

進入 container：

```
docker exec -it openwrt /bin/sh
```

---

## 7.1 `/etc/config/network`

```
config interface 'loopback'
    option ifname 'lo'
    option proto 'static'
    option ipaddr '127.0.0.1'
    option netmask '255.0.0.0'

config interface 'lan'
    option type 'bridge'
    option ifname 'eth1 eth2 eth3 eth4 eth5 eth6 eth7'
    option proto 'static'
    option ipaddr '192.168.1.1'
    option netmask '255.255.255.0'

config interface 'hostlink'
    option ifname 'veth-openwrt'
    option proto 'static'
    option ipaddr '10.0.0.1'
    option netmask '255.255.255.0'

config interface 'wan'
    option ifname 'eth8'
    option proto 'dhcp'
```

---

## 7.2 `/etc/config/firewall`

```
config zone
    option name 'lan'
    list network 'lan'
    list network 'hostlink'
    option input 'ACCEPT'
    option output 'ACCEPT'
    option forward 'ACCEPT'

config zone
    option name 'wan'
    list network 'wan'
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'REJECT'
    option masq '1'
    option mtu_fix '1'

config forwarding
    option src 'lan'
    option dest 'wan'
```

---

# ✅ 完成

你現在擁有：

- Host 完全不使用任何實體 NIC
- 所有實體 NIC 完全由 OpenWrt 控制
- Host 所有流量經 OpenWrt firewall/NAT
- Host ↔ OpenWrt 透過 veth
- 開機自動化、可重現、可維護
