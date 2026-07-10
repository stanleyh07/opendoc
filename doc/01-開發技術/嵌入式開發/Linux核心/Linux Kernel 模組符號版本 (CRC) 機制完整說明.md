
## 1. 概念總覽

- **符號版本 (CRC)**：Linux kernel 使用 `genksyms` 工具，對每個 `EXPORT_SYMBOL` 的符號 prototype 計算一個 32-bit CRC 值。
- **目的**：確保外部模組編譯時所依賴的符號介面，與當前 kernel 匯出的符號介面一致，避免 ABI 不相容。(Kernel 開啟 `CONFIG_MODVERSIONS=y` 時啟用)

---

## 2. CRC 的生成流程

1. **工具**：`scripts/genksyms/`
2. **輸入**：符號的 prototype（函式簽名、參數型態、結構體定義、宏展開）。
3. **演算法**：CRC32，實作在 `hash.c`。
4. **輸出**：寫入 `Module.symvers`，格式如下：
    
    ```
    0xadcf987e    dev_info    kernel/printk    EXPORT_SYMBOL
    ```
    

---

## 3. 模組編譯如何帶入 CRC

1. 外部模組編譯時，Makefile 指定 kernel build system：
    
    ```make
    make -C /lib/modules/$(uname -r)/build M=$PWD
    ```
    
2. **工具**：`scripts/mod/modpost.c`
    - 讀取 `Module.symvers`。
    - 找出模組依賴的符號及其 CRC。
3. 在 `.mod.c` 檔案裡生成 `__versions` section：
    
    ```c
    static const struct modversion_info ____versions[] = {
        { 0xadcf987e, "dev_info" },
        { 0x53dfe382, "module_layout" },
    };
    ```
    
4. 編譯完成後，這些 CRC 被打包進 `.ko`。

---

## 4. 模組載入時的檢查

1. **工具**：kernel module loader (`kernel/module.c`)
2. **流程**：
    - 讀取 `.ko` 裡的 `__versions` section。
    - 對每個符號，比對 kernel 當前匯出的 CRC 與 `.ko` 裡的 CRC。
    - 一致 → 載入成功；不同 → 報錯：
        
        ```
        disagrees about version of symbol dev_info
        ```
        

---

## 5. 影響範圍邏輯

- **CRC 是獨立計算的**：每個符號 prototype 各自有一個 CRC。
- **模組只比對自己依賴的符號**：沒依賴的符號即使 CRC 改了也不影響。
- **什麼會改變 CRC**：
    - 函式簽名改變。
    - 結構體定義改變（增減欄位、型態不同）。
    - 宏或 inline function 展開不同（受 `CONFIG_*` 控制）。
- **什麼不會改變 CRC**：
    - 函式內部實作改變（演算法、log、變數）。
    - CONFIG 改動但不牽動 prototype。

---

## 6. 具體例子

### 影響 CRC 的例子

```c
struct device {
    struct device *parent;
    struct kobject kobj;
    const char *init_name;   // 新增欄位
};
```

→ `_dev_info(const struct device *dev, ...)` 的 prototype 改變 → CRC 改 → NVIDIA driver mismatch。

### 不影響 CRC 的例子

```c
int add_numbers(int a, int b)
{
    printk(KERN_DEBUG "sum=%d\n", a+b);
    return a+b;
}
EXPORT_SYMBOL(add_numbers);
```

→ prototype 沒改 → CRC 相同 → 外部模組不受影響。

---

## 7. CONFIG 的影響範例

- `CONFIG_WIREGUARD_DEBUG=y`
    - 只是增加 debug log，不改 prototype → CRC 不變 → 不影響外部模組。
- `CONFIG_WIREGUARD_CRYPTO=y`
    - 牽動 crypto struct 定義 → prototype 改 → CRC 改 → 依賴這些符號的模組受影響。

---

## 8. 總結

- **CRC 是 prototype 雜湊**，只要 prototype 沒改，CRC 就一樣。
- **模組只比對自己依賴的符號**，所以「不相關就不影響」。
- **看似全部都變的情況**，其實是因為某個全域 struct 或宏改動牽連到大量 API，導致很多符號 CRC 改 → 很多模組受影響。

---

✅ **一句話收斂**：  
Linux kernel 的符號版本機制透過 CRC 確保 ABI 相容性。只有符號 prototype 改變才會導致 CRC 改變，模組只會受影響於它依賴的符號；不相關的符號即使 CRC 改了也不影響。
這裡是完整的 **符號 CRC 影響矩陣表**，用 Markdown 呈現，讓你一眼看清楚不同改動類型對 CRC 與模組的影響：

# 符號 CRC 影響矩陣

|改動類型|CRC 是否改變|受影響範圍|範例|
|---|---|---|---|
|**函式內部實作**（演算法、log、變數）|不改變|無影響|`int foo(int a)` 裡面多加 `printk()`，CRC 不變|
|**函式簽名**（參數數量/型態/返回值）|改變|依賴該函式的模組|`int foo(int a)` → `int foo(int a, int b)`|
|**結構體定義**（增減欄位、型態改變）|改變|所有用到該 struct 的函式 → 依賴這些函式的模組|`struct device` 多一個欄位 → `_dev_info()` CRC 改變|
|**typedef / 宏展開不同**|改變|用到該 typedef/宏的函式 → 依賴這些函式的模組|`typedef unsigned long size_t` → `typedef unsigned int size_t`|
|**inline function 展開不同**（受 CONFIG 控制）|改變|用到該 inline 的模組|`static inline foo()` 在 `CONFIG_DEBUG` 下展開不同|
|**Kconfig 改動但不牽動 prototype**|不改變|無影響|`CONFIG_WIREGUARD_DEBUG=y` → 只是增加 debug log，不改 prototype|
|**Kconfig 改動牽動共用 struct/API**|改變|所有依賴這些 API 的模組|啟用 WireGuard → 改 crypto/netlink struct → NVIDIA driver mismatch|

---

## 核心結論

- **CRC 是 prototype 雜湊**：只要 prototype 沒改，CRC 就一樣。
- **模組只比對自己依賴的符號**：沒依賴的符號即使 CRC 改了也不影響。
- **看似「全部都變」的情況**：其實是因為某個全域 struct 或宏改動牽連到大量 API，導致很多符號 CRC 改 → 很多模組受影響。
