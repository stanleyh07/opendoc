# 基本功能

1. **在設備樹中添加 PCA9555 擴展 GPIO**
    - **設備樹節點添加示例**
        - 首先，要在設備樹裡為 PCA9555 芯片新增一個節點。假設 PCA9555 是透過 I2C 接口連接，I2C 設備位址是`0x20`（這只是示例位址）。以下是可能的設備樹節點示例：
 
```
&i2c1 {
    pca9555: pca9555@20 {
        compatible = "nxp,pca9555";
        reg = <0x20>;
        gpio - controller;
        #gpio - cells = <2>;
    };
}
```

   在這個例子中，`&i2c1`代表設備連接到 I2C1 總線。`pca9555: pca9555@20`是節點名稱，其中`@20`是設備在 I2C 總線上的位址。`compatible`屬性指定了設備的驅動兼容字串，`reg`屬性設定了設備的寄存器位址（對於 I2C 設備而言，就是 I2C 設備位址）。`gpio - controller`屬性表示這個設備是一個 GPIO 控制器，`#gpio - cells`屬性表示每個 GPIO 描述符使用的單元數量，對於 PCA9555 通常是 2 個（一個用於端口號，一個用於引腳號）。
   
- **修改驅動以支援擴展 GPIO**
    - 驅動程式需要能識別設備樹中的擴展 GPIO 節點。以 Linux 內核為例，在註冊 GPIO 控制器時，會使用設備樹中的資訊。對於 PCA9555 驅動，它會解析`#gpio - cells`等屬性來正確配置 GPIO。
    - 在内核驅動程式碼中，會有類似以下的程式碼片段用於註冊 GPIO 控制器：

```
#include <linux/gpio/driver.h>
#include <linux/module.h>
#include <linux/of.h>
#include <linux/of_gpio.h>
#include <linux/i2c.h>

// 假設這是PCA9555驅動的結構體
struct pca9555_data {
    struct i2c_client *client;
    struct gpio_chip chip;
};

// 註冊GPIO控制器函數
static int pca9555_gpio_probe(struct i2c_client *client,
                              const struct i2c_device_id *id)
{
    struct pca9555_data *data;
    int ret;
    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    data->client = client;
    data->chip.label = dev_name(&client->dev);
    data->chip.owner = THIS_MODULE;
    data->chip.dev = &client->dev;
    data->chip.ngpio = 16;  // PCA9555有16個GPIO引腳
    data->chip.base = -1;
    data->chip.of_node = client->dev.of_node;
    data->chip.direction_input = pca9555_gpio_direction_input;
    data->chip.direction_output = pca9555_gpio_direction_output;
    data->chip.get = pca9555_gpio_get;
    data->chip.set = pca9555_gpio_set;
    data->chip.of_gpio_n_cells = 2;
    // 註冊GPIO控制器
    ret = gpiochip_add_data(&data->chip, data);
    if (ret < 0) {
        dev_err(&client->dev, "Failed to register GPIO chip\n");
        return ret;
    }
    return 0;
}
```
  

- 這段程式碼是簡化的 PCA9555 GPIO 控制器驅動的探測函數。它分配了驅動數據結構，設定了 GPIO 芯片的各種屬性，如標籤、所有者、設備指針、GPIO 引腳數量、基本引腳號等。其中`chip.of_gpio_n_cells`設定為 2，與設備樹中的`#gpio - cells`屬性匹配。然後透過`gpiochip_add_data`函數註冊 GPIO 控制器。
  

2. **其他驅動使用擴展 GPIO 的設定**
    - **在其他設備樹節點中引用擴展 GPIO**
        - 假設有另一個設備（比如 LED 設備）需要使用 PCA9555 的 GPIO 來控制。在 LED 設備的設備樹節點中，可以這樣引用：
 

```
led_device {
    compatible = "mycompany,led";
    gpios = <&pca9555 0 1>;  // 使用PCA9555的第0端口第1引腳
};
```

  
- 這裡`gpios`屬性的值`<&pca9555 0 1>`表示引用`pca9555`這個 GPIO 控制器，第一個數字`0`表示端口號（依據 PCA9555 的寄存器佈局確定），第二個數字`1`表示引腳號。
- **在使用擴展 GPIO 的驅動中獲取 GPIO 號**
    - 在 LED 驅動程式中，要獲取這個 GPIO 號並進行操作。通常會使用`of_get_gpio`函數來獲取 GPIO 號。例如：


```
#include <linux/of_gpio.h>

// 在LED驅動的探測函數中
int led_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    int gpio;
    gpio = of_get_gpio(dev->of_node, 0);
    if (gpio < 0) {
        dev_err(dev, "Failed to get GPIO\n");
        return gpio;
    }
    // 然後可以使用gpio_request等函數來請求和配置GPIO
    return 0;
}
```
  
- 這段程式碼在 LED 驅動的探測函數中，透過`of_get_gpio`函數獲取設備樹中指定的 GPIO 號。參數`0`表示`gpios`屬性中的第一個 GPIO 描述（如果有多個）。然後可以使用`gpio_request`等函數來請求和配置這個 GPIO 用於控制 LED。 

3. **關於`/sys/class/gpio`中引腳號的計算**
    - **本地 GPIO 和擴展 GPIO 的區別**
        - 對於 SoC（片上系統）本身的 GPIO，它們通常有固定的編號範圍，這些編號直接對應芯片的 GPIO 引腳。例如，一個 SoC 可能有 GPIO0 - GPIO100，這些編號是由芯片設計決定的。
    - **擴展 GPIO 在`/sys/class/gpio`中的編號方式**
        - 對於擴展 GPIO，如 PCA9555 的 GPIO，它們在`/sys/class/gpio`中的編號是相對獨立的。一般來說，當擴展 GPIO 控制器被正確註冊後，內核會為其分配一個範圍的 GPIO 編號。
        - 假設 SoC 本身的 GPIO 編號到 100 結束，而 PCA9555 有 16 個 GPIO。那內核可能會從 101 開始為 PCA9555 的 GPIO 分配編號，也就是編號範圍是 101 - 116。具體的編號分配是由內核的 GPIO 子系統根據註冊順序和可用編號空間來確定的。在設備樹中引用擴展 GPIO 時，驅動會透過設備樹中的資訊，把這些相對編號轉換成對擴展 GPIO 控制器的實際操作，例如讀取或寫入 PCA9555 的寄存器來控制相應的引腳。
        - 當驅動程式加載後，PCA9555 的 GPIO 引腳會在 `/sys/class/gpio` 中顯示為一個新的 gpiochip。假設這個 gpiochip 的基礎號碼是 504，那麼第 5 個引腳的全局 GPIO 編號就是 504 + 5 = 509


# 中斷機制

1. **擴展 GPIO 中斷功能的基本概念**    
    - 以 PCA9555 為例，它的中斷功能允許外部事件（如引腳狀態變化）觸發中斷請求。當中斷發生時，系統可以及時響應，例如在有按鈕連接到 PCA9555 引腳時，按下按鈕就能觸發中斷，進而執行相應的操作。
2. **在擴展 GPIO 控制器（PCA9555）端的設定**    
    - **設備樹設定**
        - 首先，在設備樹中需要啟用中斷功能。假設 PCA9555 的中斷線連接到 SoC 的中斷控制器的某個中斷引腳（例如中斷號為`IRQ10`），以下是修改後的設備樹節點示例：

```
&i2c1 {
    pca9555: pca9555@20 {
        compatible = "nxp,pca9555";
        reg = <0x20>;
        gpio - controller;
        #gpio - cells = <2>;
        interrupt - controller;
        #interrupt - cells = <2>;
        interrupt - parent = <&gpio1>;  // 假設中斷父節點是gpio1（通用中斷控制器）
        interrupts = <17 IRQ_TYPE_EDGE_RISING>;  // 中斷號17，觸發類型上升沿觸發，具體觸發類型依據需求和硬件而定）
    };
}
```
  
- 新增的`interrupt - controller`表明此設備是一個中斷控制器。`#interrupt - cells`定義了中斷描述符的單元數量，通常是 2（一個用於中斷號，一個用於中斷觸發類型）。`interrupt - parent`指定了中斷的父節點，`interrupts`則具體定義了中斷號和觸發類型。
    
- **驅動程式設定**
    
    - 在 PCA9555 的驅動程式中，需要註冊中斷處理函數。以下是一個簡化的示例片段：

```
#include <linux/interrupt.h>
#include <linux/gpio/driver.h>
#include <linux/module.h>
#include <linux/of.h>
#include <linux/of_gpio.h>
#include <linux/i2c.h>

// 假設這是PCA9555驅動的結構體
struct pca9555_data {
    struct i2c_client *client;
    struct gpio_chip chip;
    int irq;
};

// 中斷處理函數
irqreturn_t pca9555_interrupt_handler(int irq, void *dev_id)
{
    struct pca9555_data *data = (struct pca9555_data *)dev_id;
    // 這裡可以添加檢查哪個GPIO引腳觸發中斷的邏輯，例如讀取PCA9555的中斷狀態寄存器
    // 假設有函數pca9555_read_interrupt_status用於讀取中斷狀態寄存器
    u8 interrupt_status = pca9555_read_interrupt_status(data->client);
    // 根據中斷狀態寄存器的值，確定觸發中斷的引腳，然後可以執行相應的操作
    for (int i = 0; i < data->chip.ngpio; i++) {
        if (interrupt_status & (1 << i)) {
            // 引腳i觸發了中斷，這裡可以添加具體的處理邏輯，例如打印信息
            printk(KERN_INFO "PCA9555 GPIO %d triggered an interrupt\n", i);
        }
    }
    return IRQ_HANDLED;
}

// 註冊GPIO控制器函數（部分修改以包含中斷註冊）
static int pca9555_gpio_probe(struct i2c_client *client,
                              const struct i2c_device_id *id)
{
    struct pca9555_data *data;
    int ret;
    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    data->client = client;
    data->chip.label = dev_name(&client->dev);
    data->chip.owner = THIS_MODULE;
    data->chip.dev = &client->dev;
    data->chip.ngpio = 16;  // PCA9555有16個GPIO引腳
    data->chip.base = -1;
    data->chip.of_node = client->dev.of_node;
    data->chip.direction_input = pca9555_gpio_direction_input;
    data->chip.direction_output = pca9555_gpio_direction_output;
    data->chip.get = pca9555_gpio_get;
    data->chip.set = pca9555_gpio_set;
    data->chip.of_gpio_n_cells = 2;
    // 獲取中斷號
    data->irq = irq_of_parse_and_map(client->dev.of_node, 0);
    if (data->irq <= 0) {
        dev_err(&client->dev, "Failed to get interrupt number\n");
        return -EINVAL;
    }
    // 註冊中斷處理函數
    ret = request_irq(data->irq, pca9555_interrupt_handler,
                      IRQ_TYPE_EDGE_RISING,
                      "pca9555 - interrupt", data);
    if (ret < 0) {
        dev_err(&client->dev, "Failed to request interrupt\n");
        return ret;
    }
    // 註冊GPIO控制器
    ret = gpiochip_add_data(&data->chip, data);
    if (ret < 0) {
        dev_err(&client->dev, "Failed to register GPIO chip\n");
        // 如果註冊GPIO控制器失敗，需要釋放已註冊的中斷
        free_irq(data->irq, data);
        return ret;
    }
    return 0;
}
```

- 在`pca9555_gpio_probe`函數中，首先通過`irq_of_parse_and_map`函數獲取中斷號。如果中斷號獲取成功，就使用`request_irq`函數註冊中斷處理函數`pca9555_interrupt_handler`。在中斷處理函數中，會讀取 PCA9555 的中斷狀態寄存器，以確定是哪個引腳觸發了中斷，然後可以執行相應的操作。

3. **其他裝置如何使用擴展 GPIO 的中斷功能**
    
    - **設備樹引用**
        - 假設另一個設備（比如有一個外部按鈕連接到 PCA9555 的 GPIO 引腳，且希望在按鈕按下時觸發中斷來通知此設備），在其設備樹節點中引用 PCA9555 的中斷功能。示例如下：

```
button_device {
    compatible = "mycompany,button";
    interrupt - parent = <&pca9555>;
    interrupts = <0 1>;  // 使用PCA9555的第0端口第1引腳的中斷，觸發類型1（例如下降沿觸發）
};
```
  
- 這裡`interrupt - parent`指定了中斷的源是 PCA9555，`interrupts`定義了使用 PCA9555 的哪個引腳的中斷以及觸發類型。
    
- **驅動程式使用中斷**
    
    - 在按鈕設備的驅動程式中，也需要註冊中斷處理函數。以下是簡化的示例：

```
#include <linux/interrupt.h>
#include <linux/of_gpio.h>
#include <linux/of_device.h>

// 假設這是按鈕驅動的結構體
struct button_data {
    struct device *dev;
    int irq;
};

// 按鈕中斷處理函數
irqreturn_t button_interrupt_handler(int irq, void *dev_id)
{
    struct button_data *data = (struct button_data *)dev_id;
    printk(KERN_INFO "Button pressed, interrupt received\n");
    // 這裡可以添加更多按鈕按下後的操作，比如通知應用層等
    return IRQ_HANDLED;
}

// 按鈕驅動的探測函數
int button_probe(struct platform_device *pdev)
{
    struct button_data *data;
    int ret;
    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    data->dev = &pdev->dev;
    // 獲取中斷號
    data->irq = of_irq_get(pdev->dev.of_node, 0);
    if (data->irq <= 0) {
        dev_err(data->dev, "Failed to get interrupt number\n");
        return -EINVAL;
    }
    // 註冊中斷處理函數
    ret = request_irq(data->irq, button_interrupt_handler,
                      IRQ_TYPE_EDGE_FALLING,
                      "button - interrupt", data);
    if (ret < 0) {
        dev_err(data->dev, "Failed to request interrupt\n");
        return ret;
    }
    return 0;
}
```

- 在`button_probe`函數中，首先通過`of_irq_get`函數獲取中斷號。如果中斷號獲取成功，就使用`request_irq`函數註冊中斷處理函數`button_interrupt_handler`。當按鈕按下觸發 PCA9555 的中斷，進而觸發此按鈕設備的中斷處理函數時，就可以在函數中執行相應的操作，如打印信息或者通知應用層按鈕事件。

需要注意的是，以上代碼是基於 Linux 內核的概念和 API 進行簡化和說明的，在實際應用中，可能需要根據具體的硬件平台、內核版本和應用場景進行適當的調整和完善。

