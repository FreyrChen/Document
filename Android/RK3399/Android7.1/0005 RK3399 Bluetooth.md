### 1. 硬件配置：
* 目前产品的蓝牙的 UART 没有分配，现在分配给COM口去了。
* 如果需要使用蓝牙需要把串口拉过去。

### 2. DTS 配置：
* rk3399-excavator-sapphire.dtsi
```
    wireless-bluetooth {
        compatible = "bluetooth-platdata";
        clocks = <&rk808 1>;
        clock-names = "ext_clock";
        //wifi-bt-power-toggle;
        uart_rts_gpios = <&gpio2 19 GPIO_ACTIVE_LOW>; /* GPIO2_C3 */
        pinctrl-names = "default", "rts_gpio";
        pinctrl-0 = <&uart0_rts>;
        pinctrl-1 = <&uart0_gpios>;
        //BT,power_gpio  = <&gpio3 19 GPIO_ACTIVE_HIGH>; /* GPIOx_xx */
        BT,reset_gpio    = <&gpio0 9 GPIO_ACTIVE_HIGH>; /* GPIO0_B1 */
        BT,wake_gpio     = <&gpio2 26 GPIO_ACTIVE_HIGH>; /* GPIO2_D2 */
        BT,wake_host_irq = <&gpio0 4 GPIO_ACTIVE_HIGH>; /* GPIO0_A4 */
        status = "okay";
    };
```

### 3. Android层配置
* hardware/broadcom/libbt/include/bt_vendor_brcm.h
```
SCO_PCM_ROUTING = 0
SCO_PCM_IF_CLOCK_RATE = 1
SCO_PCM_IF_FRAME_TYPE = 0
SCO_PCM_IF_SYNC_MODE = 1
SCO_PCM_IF_CLOCK_MODE = 1
PCM_DATA_FMT_SHIFT_MODE = 0
PCM_DATA_FMT_FILL_BITS = 0
PCM_DATA_FMT_FILL_METHOD = 0
PCM_DATA_FMT_FILL_NUM = 0
PCM_DATA_FMT_JUSTIFY_MODE = 0

// 以及配置串口名称
/* Run-time configuration file */
#ifndef VENDOR_LIB_CONF_FILE
#define VENDOR_LIB_CONF_FILE "/etc/bluetooth/bt_vendor.conf"
#endif

/* Device port name where Bluetooth controller attached */
#ifndef BLUETOOTH_UART_DEVICE_PORT
#define BLUETOOTH_UART_DEVICE_PORT      "/dev/ttyO1"    /* maguro */
#endif
```