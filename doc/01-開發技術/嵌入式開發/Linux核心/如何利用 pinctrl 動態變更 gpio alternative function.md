
以動態切換 i2s_sclktx 以及 gpio 功能為例

在 device tree 中，增加 pinctrl-1 (i2s) / pinctrl-2 (gpio) 兩組定義
```
&i2s1_8ch {
	status = "okay";
	rockchip,clk-trcm = <1>;
	rockchip,playback-only;
	pinctrl-names = "default", "i2s", "gpio";
	pinctrl-0 = <
			&i2s1m1_lrcktx
			&i2s1m1_sdi0
			&i2s1m1_sdo0>;
	pinctrl-1 = <&i2s1m1_sclktx>;
	pinctrl-2 = <&i2s1m1_sclktx_gpio>;
};

&pinctrl {
	i2s1 {
		i2s1m1_sclktx: i2s1m1-sclktx {
			rockchip,pins =
				/* i2s1m1_sclktx */
				<3 RK_PC7 4 &pcfg_pull_none>;
		};
		i2s1m1_sclktx_gpio: i2s1m1-sclktx-gpio {
			rockchip,pins =
				/* i2s1m1_sclktx */
				<3 RK_PC7 RK_FUNC_GPIO &pcfg_pull_none>;
		};
	}
}
```


在 driver probe 中讀取device tree 定義
```
// 在 driver data struct 中添加幾個變數，後續會使用到
struct rk_i2s_tdm_dev {
	.....
	struct pinctrl *pinctrl;
	struct pinctrl_state *gpio_state;
	struct pinctrl_state *i2s_state;
};

// 在 probe 中讀取 device tree 設定，紀錄到 device data struct 中
static int rockchip_i2s_tdm_probe(struct platform_device *pdev)
{
	......
	// 讀取 pinctrl node
	i2s_tdm->pinctrl = devm_pinctrl_get(&pdev->dev);
	if (IS_ERR(i2s_tdm->pinctrl)) {
		dev_err(&pdev->dev, "Failed to get pinctrl\n");
		return PTR_ERR(i2s_tdm->pinctrl);
	}

	// 透過 pinctrl node 讀取 i2s 定義
	i2s_tdm->i2s_state = pinctrl_lookup_state(i2s_tdm->pinctrl, "i2s");
	if (IS_ERR(i2s_tdm->i2s_state)) {
		dev_err(&pdev->dev, "Failed to lookup I2S state\n");
		return PTR_ERR(i2s_tdm->i2s_state);
	}

	// 透過 pinctrl node 讀取 gpio 定義
	i2s_tdm->gpio_state = pinctrl_lookup_state(i2s_tdm->pinctrl, "gpio");
	if (IS_ERR(i2s_tdm->gpio_state)) {
		dev_err(&pdev->dev, "Failed to lookup GPIO state\n");
		return PTR_ERR(i2s_tdm->gpio_state);
	}
}

// 下面展示如何切換 pin function
static int i2s_tdm_runtime_suspend(struct device *dev)
{
	struct rk_i2s_tdm_dev *i2s_tdm = dev_get_drvdata(dev);

	....
	// 切換為 gpio
	pinctrl_select_state(i2s_tdm->pinctrl, i2s_tdm->gpio_state);

	....
}

static int i2s_tdm_runtime_resume(struct device *dev)
{
	struct rk_i2s_tdm_dev *i2s_tdm = dev_get_drvdata(dev);

	.....
	// 切換為 i2s 的 i2s_sclktx
	pinctrl_select_state(i2s_tdm->pinctrl, i2s_tdm->i2s_state);

	.....
}

```

