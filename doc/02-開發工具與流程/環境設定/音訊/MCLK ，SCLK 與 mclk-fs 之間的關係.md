當涉及音頻接口時，MCLK（主時鐘）、SCLK（串行時鐘）和 `mclk-fs` 設定之間的關係是非常重要的。讓我們詳細介紹這些概念並舉例說明。

### MCLK（主時鐘）

[MCLK 是音頻系統中的主時鐘信號，通常用於為音頻編解碼器提供參考時鐘。MCLK 的頻率通常是音頻采樣率的多倍，例如 128 倍、256 倍、384 倍或 512 倍](https://blog.csdn.net/qq_34414144/article/details/115975859)[1](https://blog.csdn.net/qq_34414144/article/details/115975859)。

### SCLK（串行時鐘）

[SCLK，也稱為 BCLK（位時鐘），是用於同步數據傳輸的時鐘信號。SCLK 的頻率通常是采樣率和每個樣本位數的乘積。例如，如果采樣率是 16kHz，每個樣本 16 位，那麼 SCLK 的頻率將是 16kHz * 16 * 2 = 512kHz](https://blog.csdn.net/qq_34414144/article/details/115975859)[1](https://blog.csdn.net/qq_34414144/article/details/115975859)。

### `mclk-fs` 設定

`mclk-fs` 設定表示 MCLK 與音頻流速率（采樣率）之間的乘數。在你的例子中，`mclk-fs = <128>` 意味著 MCLK 的頻率是采樣率的 128 倍。

### 實際範例

假設我們有一個音頻系統，其采樣率為 48kHz，每個樣本 16 位。那麼：

- **MCLK**: 如果 `mclk-fs = <128>`，那麼 MCLK 的頻率將是 48kHz * 128 = 6.144MHz。
- **SCLK**: SCLK 的頻率將是 48kHz * 16 * 2 = 1.536MHz。

這樣的配置確保了音頻數據的準確傳輸和時鐘同步