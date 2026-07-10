### 如何在 kernel 5.10 中啟用 DP sound

可使用 simple-audio-card driver，但無 DP plugged event，所以須改用 rockchip_hdmi driver
使用下面 devicetree
```
   	dp_sound: dp-sound {
        status = "okay";
        compatible = "rockchip,hdmi";
        rockchip,card-name= "rockchip-hdmi1";
        rockchip,mclk-fs = <128>;
        rockchip,cpu = <&spdif>;
        rockchip,codec = <&cdn_dp>;
		rockchip,jack-det;
    };
```
   
   注意，DP sound 使用的 cpu_dai 是 spdif，因此需要啟用它，
   另外，DP sound 用到的是 SOC 內部線路，不需要外部 pin 無關，
   若外部 pin 已經有其他用途，需要將其從 devicetree 移除
   舉例，這個開發版線路將其外部 pin 作為 pcie_reset 使用，須將 spdif pin 相關設定移除
   
```
&spdif {
	status = "okay";
	/*
	 * GPIO4_C5/SPDIF_tx is routed to pcie_reset, delete it
	 */
	/delete-property/ pinctrl-0;
	/delete-property/ pinctrl-names;
};

```

   還有，spdif dma 的相關設定，
   rk3399 dma0 共有 6 組 channel ，給四組硬體 i2s0、i2s1、i2s2、spdif (每個佔 tx / rx 兩個)
   因此最多同時三組硬體使用，所以須注意是否有 dma channel 可以使用
   舉例，這個開發版啟用 i2s0、i2s2，因此需要確認 i2s1 被 disable，這樣 spdif 才可以 request 到 DMA
   
```
&i2s1 {
	status = "disabled";
};
```

   還須注意 cdn_dp 設定，原始 dai-cells 設定為 1，需設為 0，讓它自行計算，
   新 driver 會設定 tx / rx 兩組 (但只用到 tx)
   
```
&cdn_dp {
	status = "okay"; 
	#sound-dai-cells = <0>;   // 從 1 改為 0
};
```
 
注意因為機制修改，而原來 driver 是採用一般通用寫法，會使得單用 spdif 的發生錯誤

```
static int cdn_dp_audio_codec_init(struct cdn_dp_device *dp,
				   struct device *dev)
{
	struct hdmi_codec_pdata codec_data = {
		.i2s = 0,  // 從 1 改為 0，因為 DP sound 未使用 i2s，initial 會失敗
		.spdif = 1,
		.ops = &audio_codec_ops,
		.max_i2s_channels = 8,
	};
	.....
}
```

