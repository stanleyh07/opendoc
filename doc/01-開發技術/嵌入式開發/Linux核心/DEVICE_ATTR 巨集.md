在 Linux kernel 中，`DEVICE_ATTR` 巨集用於定義設備屬性，這些屬性可以通過 sysfs 文件系統導出，從而允許用戶空間程序讀寫設備屬性。這些屬性通常用於設備驅動程序中，以便在 sysfs 中創建文件，從而實現與設備的交互。

### `DEVICE_ATTR` 巨集定義

`DEVICE_ATTR` 巨集定義如下：

```c
#define DEVICE_ATTR(_name, _mode, _show, _store) \
struct device_attribute dev_attr_##_name = __ATTR(_name, _mode, _show, _store)
```

- `_name`：屬性的名稱。
- `_mode`：文件的權限模式（例如 0644 表示所有人可讀，只有擁有者可寫）。
- `_show`：指向顯示屬性值的函數指針。
- `_store`：指向設置屬性值的函數指針。

### 使用範例

以下是一個完整的範例，展示如何使用 `DEVICE_ATTR` 巨集來創建和使用設備屬性：

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/platform_device.h>
#include <linux/sysfs.h>
#include <linux/kobject.h>

static char mybuf[100] = "123";

// 顯示屬性值的函數
static ssize_t show_my_device(struct device *dev, struct device_attribute *attr, char *buf) {
    return sprintf(buf, "%s\n", mybuf);
}

// 設置屬性值的函數
static ssize_t set_my_device(struct device *dev, struct device_attribute *attr, const char *buf, size_t len) {
    snprintf(mybuf, sizeof(mybuf), "%s", buf);
    return len;
}

// 定義設備屬性
static DEVICE_ATTR(my_device_test, 0644, show_my_device, set_my_device);

static int __init my_device_init(void) {
    struct device *mydev;
    int ret;

    // 創建設備
    mydev = device_create(cls, NULL, MKDEV(major, 0), NULL, "mytest_device");
    if (IS_ERR(mydev)) {
        pr_err("Failed to create device\n");
        return PTR_ERR(mydev);
    }

    // 創建 sysfs 文件
    ret = device_create_file(mydev, &dev_attr_my_device_test);
    if (ret) {
        pr_err("Failed to create sysfs file\n");
        device_destroy(cls, MKDEV(major, 0));
        return ret;
    }

    return 0;
}

static void __exit my_device_exit(void) {
    device_remove_file(mydev, &dev_attr_my_device_test);
    device_destroy(cls, MKDEV(major, 0));
}

module_init(my_device_init);
module_exit(my_device_exit);

MODULE_LICENSE("GPL");
```

### 定義產生的內容

上述範例中，`DEVICE_ATTR` 巨集會展開為以下結構：

```c
struct device_attribute dev_attr_my_device_test = {
    .attr = {
        .name = "my_device_test",
        .mode = 0644,
    },
    .show = show_my_device,
    .store = set_my_device,
};
```

這樣，當設備創建時，會在 sysfs 中創建一個名為 `my_device_test` 的文件，並且可以通過 `cat` 命令讀取其值，通過 `echo` 命令設置其值。

