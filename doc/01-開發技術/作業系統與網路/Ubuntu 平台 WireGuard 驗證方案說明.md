
這份說明文件說明如何進行 **WireGuard 驗證**。此方案適用於 **Ubuntu 22.04** 以及 **Jetson 非標準內核**環境，能在單一主機上透過 network namespace 模擬兩台機器，驗證 WireGuard 功能。

---

## Script 最終方案

```bash
#!/usr/bin/env bash
# wireguard-test.sh
# Usage: ./wireguard-test.sh [init|check|clean|kernelcheck]

set -euo pipefail

MODE=${1:-}

WG_IF="wg0"
NS1="ns1"
NS2="ns2"

function init_env() {
    echo "[+] Checking required packages..."

    NEED_INSTALL=()

    if ! dpkg -s wireguard-tools >/dev/null 2>&1; then
        NEED_INSTALL+=("wireguard-tools")
    fi

    if ! dpkg -s iproute2 >/dev/null 2>&1; then
        NEED_INSTALL+=("iproute2")
    fi

    if [ ${#NEED_INSTALL[@]} -gt 0 ]; then
        echo "[+] Installing missing packages: ${NEED_INSTALL[*]}"
        sudo apt update
        sudo apt install -y "${NEED_INSTALL[@]}"
    else
        echo "[+] All required packages already installed"
    fi

    echo "[+] Creating namespaces..."
    sudo ip netns add $NS1 || true
    sudo ip netns add $NS2 || true

    echo "[+] Creating veth pair..."
    sudo ip link add veth1 type veth peer name veth2
    sudo ip link set veth1 netns $NS1
    sudo ip link set veth2 netns $NS2

    echo "[+] Assigning IPs..."
    sudo ip netns exec $NS1 ip addr add 10.0.0.1/24 dev veth1
    sudo ip netns exec $NS2 ip addr add 10.0.0.2/24 dev veth2
    sudo ip netns exec $NS1 ip link set veth1 up
    sudo ip netns exec $NS2 ip link set veth2 up

    echo "[+] Generating keys..."
    KEY1_PRIV=$(wg genkey)
    KEY1_PUB=$(echo $KEY1_PRIV | wg pubkey)
    KEY2_PRIV=$(wg genkey)
    KEY2_PUB=$(echo $KEY2_PRIV | wg pubkey)

    echo "[+] Configuring WireGuard..."
    cat <<EOF | sudo tee /etc/wireguard/${WG_IF}-ns1.conf >/dev/null
[Interface]
PrivateKey = $KEY1_PRIV
Address = 192.168.100.1/24
ListenPort = 51820

[Peer]
PublicKey = $KEY2_PUB
AllowedIPs = 192.168.100.2/32
Endpoint = 10.0.0.2:51821
EOF

    cat <<EOF | sudo tee /etc/wireguard/${WG_IF}-ns2.conf >/dev/null
[Interface]
PrivateKey = $KEY2_PRIV
Address = 192.168.100.2/24
ListenPort = 51821

[Peer]
PublicKey = $KEY1_PUB
AllowedIPs = 192.168.100.1/32
Endpoint = 10.0.0.1:51820
EOF

    echo "[+] Starting WireGuard..."
    sudo ip netns exec $NS1 wg-quick up /etc/wireguard/${WG_IF}-ns1.conf
    sudo ip netns exec $NS2 wg-quick up /etc/wireguard/${WG_IF}-ns2.conf

    echo "[+] Init complete."
}

function check_env() {
    echo "[+] Checking WireGuard status..."
    sudo ip netns exec $NS1 wg show
    sudo ip netns exec $NS2 wg show

    echo "[+] Interfaces in ns1:"
    sudo ip netns exec $NS1 ip addr
    echo "[+] Interfaces in ns2:"
    sudo ip netns exec $NS2 ip addr

    echo "[+] Testing connectivity..."
    sudo ip netns exec $NS1 ping -c 3 192.168.100.2 || true
    sudo ip netns exec $NS2 ping -c 3 192.168.100.1 || true

    echo "[+] Routes:"
    sudo ip netns exec $NS1 ip route
    sudo ip netns exec $NS2 ip route

    echo "[+] Check complete."
}

function clean_env() {
    echo "[+] Cleaning up..."
    sudo ip netns exec $NS1 wg-quick down /etc/wireguard/${WG_IF}-ns1.conf || true
    sudo ip netns exec $NS2 wg-quick down /etc/wireguard/${WG_IF}-ns2.conf || true

    sudo ip link delete veth1 2>/dev/null || true
    sudo ip netns del $NS1 2>/dev/null || true
    sudo ip netns del $NS2 2>/dev/null || true

    sudo rm -f /etc/wireguard/${WG_IF}-ns1.conf /etc/wireguard/${WG_IF}-ns2.conf

    echo "[+] Removing WireGuard tools..."
    sudo apt remove --purge -y wireguard-tools
    sudo apt autoremove -y

    echo "[+] Clean complete."
}

function kernel_check() {
    echo "[+] Checking kernel config for WireGuard..."

    CONFIG_FILE=""
    GREP_CMD="grep"
    if [ -f /proc/config.gz ]; then
        CONFIG_FILE="/proc/config.gz"
        GREP_CMD="zgrep"
    elif [ -f /boot/config-$(uname -r) ]; then
        CONFIG_FILE="/boot/config-$(uname -r)"
        GREP_CMD="grep"
    else
        echo "  - No kernel config file found"
        return
    fi

    REQUIRED_OPTS=("CONFIG_WIREGUARD" "CONFIG_CRYPTO_CHACHA20" "CONFIG_CRYPTO_POLY1305" \
                   "CONFIG_CRYPTO_CURVE25519" "CONFIG_CRYPTO_BLAKE2S" "CONFIG_CRYPTO_LIB_SHA256" \
                   "CONFIG_NET" "CONFIG_INET" "CONFIG_TUN")

    for opt in "${REQUIRED_OPTS[@]}"; do
        if $GREP_CMD -q "^$opt=y" $CONFIG_FILE; then
            echo "  - $opt: built-in (OK)"
        elif $GREP_CMD -q "^$opt=m" $CONFIG_FILE; then
            echo "  - $opt: module (OK)"
        else
            echo "  - $opt: MISSING"
        fi
    done

    echo "[+] Checking WireGuard availability..."
    if $GREP_CMD -q "^CONFIG_WIREGUARD=y" $CONFIG_FILE; then
        echo "  - WireGuard is built-in, no module load needed"
    elif $GREP_CMD -q "^CONFIG_WIREGUARD=m" $CONFIG_FILE; then
        if sudo modprobe wireguard 2>/dev/null; then
            echo "  - WireGuard module loaded successfully"
        else
            echo "  - WireGuard module NOT available"
        fi
    else
        echo "  - WireGuard not enabled in kernel"
    fi
}

case "$MODE" in
    init) init_env ;;
    check) check_env ;;
    clean) clean_env ;;
    kernelcheck) kernel_check ;;
    *)
        echo "Usage: $0 [init|check|clean|kernelcheck]"
        exit 1
        ;;
esac
```

---

## 使用方式

### 1. 初始化環境

建立 network namespace、veth pair、WireGuard interface：

```bash
./wireguard-test.sh init
```

### 2. 驗證功能

檢查 WireGuard 狀態、介面、路由，並測試 `ping`：

```bash
./wireguard-test.sh check
```

### 3. 檢查內核設定

確認 Jetson 或 Ubuntu 內核是否有正確的 `CONFIG_*` 支援：

```bash
./wireguard-test.sh kernelcheck
```

### 4. 清理環境

移除 WireGuard interface、namespace、套件：

```bash
./wireguard-test.sh clean
```

---

## 說明重點

- **IP 規劃**：`192.168.100.1` 與 `192.168.100.2` 是 WireGuard interface 的測試 IP，不會影響主機真實網路。
- **Endpoint 設定**：已自動填入對方的 veth IP (`10.0.0.1` / `10.0.0.2`)，確保 WireGuard peer 能互相找到。
- **Namespace 隔離**：介面只存在於 `ns1` / `ns2`，需用 `ip netns exec` 進入 namespace 才能看到。
- **Jetson 特殊情況**：若 `kernelcheck` 顯示缺少 `CONFIG_WIREGUARD` 或加密模組，需安裝 `wireguard-dkms` 或自行編譯內核。

---

這份腳本提供了 **完整、可重現、可清理**的 WireGuard 測試方案，能在單一 Ubuntu / Jetson 主機上驗證 WireGuard 功能。