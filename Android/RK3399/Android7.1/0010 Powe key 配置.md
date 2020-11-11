* ### 查看原理图
* 原理图上面是 GPIO1_B2 和 GPIO0_A5

* ### DTS 配置：
* rk3399-android.dtsi
```
        power-key {
            gpios = <&gpio1 10 GPIO_ACTIVE_LOW>;  //<&gpio0 5 GPIO_ACTIVE_LOW>
            linux,code = <116>;
            label = "power";
            gpio-key,wakeup;
        };
```