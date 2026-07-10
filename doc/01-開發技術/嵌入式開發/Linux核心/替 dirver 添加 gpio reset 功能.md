以 `usbdrd_dwc3` 為例
當你想在 Linux 設備樹中為 `usbdrd_dwc3` 節點添加一個 GPIO 重置時，可以按照以下步驟進行：

1. **定義 GPIO 重置控制器**： 在設備樹中定義一個 GPIO 重置控制器節點，例如：
    
    ```dts
    gpio_reset: gpio-reset {
        compatible = "gpio-reset";
        reset-gpios = <&gpio5 0 GPIO_ACTIVE_LOW>;
        reset-delay-us = <10000>;
        #reset-cells = <0>;
    };
    ```
    
2. **在 `resets` 屬性中添加 GPIO 重置**： 在 `usbdrd_dwc3` 節點的 `resets` 屬性中添加新的 GPIO 重置控制器的引用，例如：
    
    ```dts
    usbdrd_dwc3: dwc3@fcc00000 {
        compatible = "snps,dwc3";
        reg = <0x0 0xfcc00000 0x0 0x400000>;
        interrupts = <GIC_SPI 169 IRQ_TYPE_LEVEL_HIGH>;
        dr_mode = "otg";
        phys = <&u2phy0_otg>, <&combphy0_us PHY_TYPE_USB3>;
        phy-names = "usb2-phy", "usb3-phy";
        phy_type = "utmi_wide";
        power-domains = <&power RK3568_PD_PIPE>;
        resets = <&cru SRST_USB3OTG0>, <&gpio_reset>;
        reset-names = "usb3-otg", "gpio-reset";
        snps,dis_enblslpm_quirk;
        snps,dis-u1-entry-quirk;
        snps,dis-u2-entry-quirk;
        snps,dis-u2-freeclk-exists-quirk;
        snps,dis-del-phy-power-chg-quirk;
        snps,dis-tx-ipgap-linecheck-quirk;
        snps,dis_rxdet_inp3_quirk;
        snps,xhci-trb-ent-quirk;
        snps,parkmode-disable-ss-quirk;
        quirk-skip-phy-init;
        status = "disabled";
    };
    ```
    

負責處理 `gpio-reset` 的驅動程式源代碼位於 Linux 內核的 `drivers/reset/reset-gpio.c` 文件中。你可以在 [GitHub](https://github.com/torvalds/linux/blob/master/drivers/reset/reset-gpio.c)) 上查看這個文件