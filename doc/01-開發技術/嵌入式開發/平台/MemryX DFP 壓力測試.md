## 🟢 完整步驟：單顆 MemryX DFP 壓力測試

### 1️⃣ 下載範例模型

MemryX 官方 GitHub 提供現成模型與程式碼：

👉 [MemryX_eXamples GitHub](https://github.com/memryx/MemryX_eXamples/tree/release)

下載方式：

```bash
git clone https://github.com/memryx/MemryX_eXamples.git
cd MemryX_eXamples
```

範例模型位置：

- `models/resnet50.onnx` → 影像分類
- `models/yolov7-tiny.onnx` → 物件偵測

---

### 2️⃣ 編譯模型成 `.dfp`

使用 **MemryX Neural Compiler** 將 ONNX 模型轉換成 DFP 檔案。

👉 [Neural Compiler 教學](https://developer.memryx.com/tools/neural_compiler.html)

範例指令：

```bash
# 編譯 ResNet50
mxnc compile --model models/resnet50.onnx --output resnet50.dfp

# 編譯 YOLOv7-tiny
mxnc compile --model models/yolov7-tiny.onnx --output yolov7-tiny.dfp
```

完成後會得到 `.dfp` 檔案，這就是 DFP 晶片能執行的模型。

---

### 3️⃣ 撰寫 Stress Test 程式

建立 `stress_test.py`，內容如下：

```python
import time
import argparse
import itertools
from memryx import DFP

def preprocess_input(input_path):
    # 簡化處理，實際可依模型範例修改
    return [0.0] * 224 * 224 * 3

def postprocess_output(result):
    # 依模型範例修改，例如取分類結果或 YOLO box
    return result

def run_stress_test(model_path, input_path, iterations, infinite, log_interval, warmup):
    # 載入模型
    dfp = DFP(model_path)
    input_data = preprocess_input(input_path)

    # 暖機
    for _ in range(warmup):
        _ = dfp.run(input_data)

    # 壓力測試
    start_time = time.time()
    count = 0

    if infinite:
        for i in itertools.count():
            result = dfp.run(input_data)
            _ = postprocess_output(result)
            count += 1
            if count % log_interval == 0:
                elapsed = time.time() - start_time
                print(f"[{count}] elapsed={elapsed:.2f}s, avg={(elapsed/count):.6f}s")
    else:
        for i in range(iterations):
            result = dfp.run(input_data)
            _ = postprocess_output(result)
            count += 1
            if count % log_interval == 0:
                elapsed = time.time() - start_time
                print(f"[{count}] elapsed={elapsed:.2f}s, avg={(elapsed/count):.6f}s")

        end_time = time.time()
        total = end_time - start_time
        print(f"Total time: {total:.2f}s")
        print(f"Average per inference: {(total/iterations):.6f}s")
        print(f"Throughput: {(iterations/total):.2f} inferences/sec")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--model", type=str, required=True, help="編譯好的 .dfp 模型檔路徑")
    parser.add_argument("--input", type=str, required=True, help="測試影像或資料路徑")
    parser.add_argument("--iterations", type=int, default=10000, help="執行次數")
    parser.add_argument("--infinite", action="store_true", help="是否無窮迴圈")
    parser.add_argument("--log-interval", type=int, default=100, help="多久輸出一次狀態")
    parser.add_argument("--warmup", type=int, default=50, help="暖機次數")
    args = parser.parse_args()

    run_stress_test(args.model, args.input, args.iterations, args.infinite, args.log_interval, args.warmup)
```

---

### 4️⃣ 執行 Stress Test

在 console 執行：

- **固定次數測試**（例如 10,000 次推論）：
    
    ```bash
    python stress_test.py --model resnet50.dfp --input test.jpg --iterations 10000
    ```
    
- **無窮迴圈測試**（直到手動停止）：
    
    ```bash
    python stress_test.py --model yolov7-tiny.dfp --input test.jpg --infinite
    ```
    
- **中斷測試**：在 console 按 `Ctrl+C` 停止。
    

---

### 5️⃣ 監控資源

- **CPU/GPU 使用率**：`top` 或 `htop`
- **溫度/功耗**：Jetson 用 `tegrastats`，x86 用 `lm-sensors`
- **DFP 狀態**：MemryX SDK log 或 MXA-Manager 工具（可在 [Testing with Fewer Chips 教學](https://developer.memryx.com/tutorials/advanced/test_with_fewer_chips.html) 找到使用方式）

---

📌 **完整流程總結**：

1. 到 [MemryX_eXamples GitHub](https://github.com/memryx/MemryX_eXamples/tree/release) 下載模型
2. 用 [Neural Compiler](https://developer.memryx.com/tools/neural_compiler.html) 編譯成 `.dfp`
3. 建立 `stress_test.py` 程式碼
4. 在 console 執行 `python stress_test.py`，可選固定次數或無窮迴圈
5. 搭配系統監控工具觀察效能與穩定性

---

要不要我幫你直接補上 **ResNet50 Classification 範例的前處理與後處理程式碼**，讓你可以直接跑而不用自己修改？