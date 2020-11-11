### 1. DTS
* 主要是配置一下功放使能 PIN.
```
&i2c1 {
    status = "okay";
    i2c-scl-rising-time-ns = <300>;
    i2c-scl-falling-time-ns = <30>;
    es8316: es8316@10 {
        #sound-dai-cells = <0>;
        compatible = "everest,es8316";
        reg = <0x10>;
        clocks = <&cru SCLK_I2S_8CH_OUT>;
        clock-names = "mclk";
        pinctrl-names = "default";
        pinctrl-0 = <&i2s_8ch_mclk>;
        pinctrl-1 = <&phone_ctl_pin>;
        spk-con-gpio = <&gpio2 12 GPIO_ACTIVE_HIGH>;
    };
};
    phone_ctl {
        phone_ctl_pin: phone_ctl_pin {
            rockchip,pins =
                <2 12  RK_FUNC_GPIO &pcfg_pull_up>;
                /* GPIO2_B4  PHONE_CTL 1*8+4 == 12 */
        };
    };
    // GPIO2_B4 是  PHONE_CTL pin  角，控制功放使能
```

### 2. sound/soc/codecs/es8316.c
* 因为这次没有耳机侦测 PIN，所以需要配置声音一直输出，修改驱动代码
```
@@ -1135,7 +1135,9 @@ static int es8316_i2c_probe(struct i2c_client *i2c,
        es8316->debounce_time = 200;
        es8316->hp_det_invert = 0;
        es8316->pwr_count = 0;
-       es8316->hp_inserted = false;
+       /* es8316->hp_inserted = false; */
+       es8316->hp_inserted = true;
```