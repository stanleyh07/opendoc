好的 👍 我幫你設計一個 **多功能 Bash script**，可以透過參數來執行三種模式：

- **init** → 初始化 fwmark 路由設定
- **check** → 檢查目前設定是否生效
- **clean** → 清除測試規則並回復原狀

---

## Script 範例

```bash
#!/bin/bash
# fwmark routing test script
# 用法: ./fwmark.sh init <gateway> | check | clean

TABLE_ID=100
TABLE_NAME="testtable"
MARK=1

function init_fwmark() {
    GATEWAY=$1
    if [ -z "$GATEWAY" ]; then
        echo "請提供 gateway，例如: ./fwmark.sh init 192.168.1.1"
        exit 1
    fi

    # 加入路由表 (若不存在)
    grep -q "$TABLE_ID $TABLE_NAME" /etc/iproute2/rt_tables || \
        echo "$TABLE_ID $TABLE_NAME" >> /etc/iproute2/rt_tables

    # 設定路由與規則
    ip route add default via $GATEWAY table $TABLE_NAME 2>/dev/null
    ip rule add fwmark $MARK table $TABLE_NAME 2>/dev/null

    # iptables 打上 mark
    iptables -t mangle -A OUTPUT -p tcp --dport 80 -j MARK --set-mark $MARK

    echo "[OK] 已初始化 fwmark 路由，gateway=$GATEWAY"
}

function check_fwmark() {
    echo "=== ip rule ==="
    ip rule show | grep fwmark || echo "未找到 fwmark 規則"

    echo "=== ip route (table $TABLE_NAME) ==="
    ip route show table $TABLE_NAME

    echo "=== 測試路由查詢 ==="
    ip route get 8.8.8.8 mark $MARK 2>/dev/null || echo "查詢失敗"
}

function clean_fwmark() {
    ip rule del fwmark $MARK table $TABLE_NAME 2>/dev/null
    iptables -t mangle -F
    ip route flush table $TABLE_NAME
    echo "[OK] 已清除 fwmark 路由設定"
}

case "$1" in
    init)
        init_fwmark $2
        ;;
    check)
        check_fwmark
        ;;
    clean)
        clean_fwmark
        ;;
    *)
        echo "用法: $0 {init <gateway>|check|clean}"
        ;;
esac
```

---

## 使用方式

1. **初始化**
    
    ```bash
    sudo ./fwmark.sh init 192.168.1.1
    ```
    
    → 建立 `testtable` 路由表，並設定 fwmark 規則。
    
2. **檢查**
    
    ```bash
    sudo ./fwmark.sh check
    ```
    
    → 顯示目前 fwmark 規則與路由狀態。
    
3. **清除**
    
    ```bash
    sudo ./fwmark.sh clean
    ```
    
    → 移除 fwmark 規則、清空 iptables mark 與路由表。
    

