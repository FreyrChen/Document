### 1. DTS 配置
```
&u2phy0 {
    status = "okay";
    extcon = <&fusb0>;
    u2phy0_otg: otg-port {
        status = "okay";
    };
    // 这些 host 必须加，不然会影响 usb2.0 host 功能。
    u2phy0_host: host-port {
        status = "okay";
    };
};

&u2phy1 {
    status = "okay";

    u2phy1_host: host-port {
        status = "okay";
    };

    // 不加影响 usb2.0 host 功能。具体原因暂时不清楚。
    u2phy1_host: host-port {
        status = "okay";
    };
};

&tcphy0 {
    extcon = <&fusb0>;
    status = "okay";
};

&tcphy1 {
    status = "okay";
};

&usbdrd3_0 {
    status = "okay";
    extcon = <&fusb0>;
};

&usbdrd3_1 {
    status = "okay";
};

&usbdrd_dwc3_0 {
    status = "okay";
    dr_mode = "otg";
    maximum-speed = "high-speed";
};

&usbdrd_dwc3_1 {
    status = "okay";
    dr_mode = "host";
};

&usb_host0_ehci {
    status = "okay";
};

&usb_host0_ohci {
    status = "okay";
};

&usb_host1_ehci {
    status = "okay";
};

&usb_host1_ohci {
    status = "okay";
};

&i2c4 {
    status = "okay";
    i2c-scl-rising-time-ns = <475>;
    i2c-scl-falling-time-ns = <26>;

    fusb0: fusb30x@22 {
        compatible = "fairchild,fusb302";
        reg = <0x22>;
        pinctrl-names = "default";
        pinctrl-0 = <&fusb0_pins>;
        int-n-gpios = <&gpio1 2 GPIO_ACTIVE_HIGH>;
        vbus-5v-gpios = <&gpio4 28 GPIO_ACTIVE_HIGH>;
        status = "okay";
    };
};

// pin mux 配置
    fusb30x {
        fusb0_pins: fusb0-pins {
            rockchip,pins =
            <1 2 RK_FUNC_GPIO &pcfg_pull_up>,
            <4 28 RK_FUNC_GPIO &pcfg_pull_up>;
        };
    };
```

### 2. 调试过程中的异常
* typec 加入 typec 数据线后一直出现 fusb diconnect 提示
* 原因是因为数据线质量不行。
* 换一个质量好的 typec 数据线即不会出现相关错误。
* 参考链接：
* https://blog.csdn.net/m0_38022615/article/details/109090478
* 错误提示如下： 
```
fusb302 4-0022: connection has disconnected
```