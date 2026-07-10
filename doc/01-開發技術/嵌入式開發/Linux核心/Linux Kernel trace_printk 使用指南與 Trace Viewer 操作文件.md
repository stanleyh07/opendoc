
# Linux Kernel `trace_printk()` 完整使用指南

（含 ftrace、debugfs、KernelShark、UART driver 實務範例）

---

## 1. 簡介

`trace_printk()` 是 Linux kernel 內建的高效能 tracing API，用來在 **不經過 printk / console / UART driver** 的情況下輸出 debug 訊息。  
它特別適合：

- UART driver（避免 recursion）
- interrupt handler
- scheduler / mm / low-level I/O path
- 任何不能呼叫 printk 的地方

`trace_printk()` 的輸出會寫入 **ftrace ring buffer**，可透過 debugfs 讀取。

---

## 2. ftrace 是什麼？

**ftrace 不是指令，也不是可執行檔。**  
它是 Linux kernel 內建的 tracing 子系統，所有控制介面都以「檔案」形式存在於：

```
/sys/kernel/debug/tracing/
```

你用 `echo` 與 `cat` 操作它，而不是執行某個 binary。

---

## 3. debugfs 是什麼？

debugfs 是 kernel 的 debug filesystem。  
ftrace 的所有介面都放在 debugfs 裡，因此你必須先掛載 debugfs 才能使用 ftrace。

掛載方式：

```bash
mount -t debugfs none /sys/kernel/debug
```

掛載後會看到：

```
/sys/kernel/debug/tracing/
```

這就是 ftrace 的操作介面。

---

## 4. Kernel 必要 CONFIG

以下選項必須啟用：

```
CONFIG_FTRACE=y
CONFIG_TRACING=y
CONFIG_TRACE_PRINTK=y
CONFIG_DEBUG_FS=y
```

建議啟用（提供更完整 tracing 能力）：

```
CONFIG_FUNCTION_TRACER=y
CONFIG_FUNCTION_GRAPH_TRACER=y
CONFIG_RING_BUFFER=y
CONFIG_EVENT_TRACING=y
CONFIG_CONTEXT_SWITCH_TRACER=y
```

檢查方式：

```bash
grep TRACE_PRINTK /boot/config-$(uname -r)
grep DEBUG_FS /boot/config-$(uname -r)
```

---

## 5. 在 Driver 中使用 `trace_printk()`

### 5.1 基本用法

```c
#include <linux/ftrace.h>

trace_printk("hello trace\n");
```

### 5.2 UART driver 實務範例（避免 recursion）

```c
#include <linux/ftrace.h>

static void my_uart_tx(struct uart_port *port, u8 val)
{
    trace_printk("serial_out_msg: port=%p val=0x%02x\n", port, val);

    /* 真正的 TX 寫入 */
    serial_out(port, UART_TX, val);
}
```

### 5.3 特性

- 不會呼叫 printk
- 不會走 console
- 不會 recursion
- 效能高於 printk
- 訊息寫入 ftrace ring buffer

---

## 6. 使用 ftrace 查看 trace_printk 輸出（不需要 trace-cmd）

許多嵌入式 Linux（BusyBox、Yocto、OpenWrt）沒有 trace-cmd。  
但你仍然可以完整使用 trace_printk。

### 6.1 即時查看（最推薦）

```bash
cat /sys/kernel/debug/tracing/trace_pipe
```

輸出範例：

```
<idle>-0     [001] ....  123.456789: trace_printk: serial_out_msg: port=ffff000012345000 val=0x41
```

### 6.2 查看完整 buffer

```bash
cat /sys/kernel/debug/tracing/trace
```

### 6.3 清空 buffer

```bash
echo > /sys/kernel/debug/tracing/trace
```

### 6.4 啟用 / 停止 tracing

啟用：

```bash
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

停止：

```bash
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

---

## 7. 使用 KernelShark（Trace Viewer GUI）

即使沒有 trace-cmd，你仍然可以使用 KernelShark。

### 7.1 安裝（Ubuntu）

```bash
sudo apt install kernelshark
```

### 7.2 匯出 trace buffer

```bash
cat /sys/kernel/debug/tracing/trace > trace.txt
```

### 7.3 在 PC 上開啟

```bash
kernelshark trace.txt
```

KernelShark 會顯示：

- CPU timeline
- 每個 task 的事件
- trace_printk() 事件

---

## 8. 最小可重現測試流程（MRE）

1. 在 driver 加入：

```c
trace_printk("TX=0x%02x\n", val);
```

2. 掛載 debugfs：

```bash
mount -t debugfs none /sys/kernel/debug
```

3. 即時查看：

```bash
cat /sys/kernel/debug/tracing/trace_pipe
```

4. 驗證輸出：

```
TX=0x41
TX=0x42
```

---

## 9. 常見問題（FAQ）

### Q1. 為什麼 trace_printk 不會 recursion？

因為它不走 printk，不會觸發 console，也不會呼叫 UART driver。

---

### Q2. 為什麼看不到 trace_printk 的輸出？

請確認：

- debugfs 已掛載
- `CONFIG_TRACE_PRINTK=y`
- tracing 已啟用：`echo 1 > tracing_on`
- 使用 trace_pipe 查看

---

### Q3. trace_printk 會影響效能嗎？

比 printk 快很多，但仍有 overhead。  
若需要更低 overhead，建議改用 tracepoints。

---

## 10. 完整結論

- **ftrace 不是指令，是 kernel tracing 子系統**
- **debugfs 是 ftrace 的介面所在位置**
- **trace_printk() 是最安全的 UART driver debug 方法**
- **不需要 trace-cmd**
- **trace_pipe 是最即時、最簡單的查看方式**
- **KernelShark 可視覺化 trace timeline**
