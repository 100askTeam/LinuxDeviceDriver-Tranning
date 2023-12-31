# 使用spidev操作SPI_OLED模块 #

参考资料：

* 内核驱动：`drivers\spi\spidev.c`

* 内核提供的测试程序：`tools\spi\spidev_fdx.c`

* 内核文档：`Documentation\spi\spidev`	

* OLEDC芯片手册：`SPEC UG-2864TMBEG01 .pdf`、`SSD1306-Revision 1.1 (Charge Pump).pdf`

* 所参考的代码在另一个GIT仓库里：`https://e.coding.net/weidongshan/01_all_series_quickstart.git`
  ![](pic/68_oled_bare_code.png)
  
  

## 1. 要做的事情

![image-20220314163549210](pic/66_3_thing_to_use_spidev.png)





## 2. 编写设备树

无论是使用IMX6ULL开发板还是STM32MP157开发板，都有类似的扩展板。把OLED模块接到扩展板的SPI_A插座上，如下：

![image-20220314115446373](pic/50_oled_on_extend_brd.png)

### 2.1.1 IMX6ULL

![](pic/44_imx6ull_pro_extend_spi_a.png)

DC引脚使用GPIO4_20，不过只需要在APP里直接控制DC引脚，无需在设备树里指定。

设备树如下：

```shell
&ecspi1 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_ecspi1>;

    fsl,spi-num-chipselects = <2>;
    cs-gpios = <&gpio4 26 GPIO_ACTIVE_LOW>, <&gpio4 24 GPIO_ACTIVE_LOW>;
    status = "okay";

    oled: oled {
        compatible = "spidev";
        reg = <0>;
        spi-max-frequency = <10000000>;
    };
};
```





### 2.1.2 STM32MP157

![](pic/45_stm32mp157_pro_extend_spi_a.png)

DC引脚使用GPIOA_13，不过只需要在APP里直接控制DC引脚，无需在设备树里指定。

设备树如下：

```shell
&spi5 {
        pinctrl-names = "default", "sleep";
        pinctrl-0 = <&spi5_pins_a>;
        pinctrl-1 = <&spi5_sleep_pins_a>;
        status = "okay";
        cs-gpios = <&gpioh 5 GPIO_ACTIVE_LOW>, <&gpioz 4 GPIO_ACTIVE_LOW>;
        spidev: icm20608@0{
                compatible = "invensense,icm20608";
                interrupts = <0 IRQ_TYPE_EDGE_FALLING>;
                interrupt-parent = <&gpioz>;
                spi-max-frequency = <8000000>;
                reg = <0>;
        };
        oled: oled@1{
                compatible = "spidev";
                spi-max-frequency = <10000000>;
                reg = <1>;
        };
};
```





## 3. 编写APP

### 3.1 怎么控制DC引脚

查看原理图，确认引脚，然后参考如下文档即可使用APP操作GPIO，假设引脚号码为100，C语言代码如下：

```c
system("echo 100  > /sys/class/gpio/export");
system("echo out > /sys/class/gpio/gpio100/direction");
system("echo 1 > /sys/class/gpio/gpio100/value");
system("echo 0 > /sys/class/gpio/gpio100/value");
system("echo 100 > /sys/class/gpio/unexport");
```

![image-20220314153844190](pic/65_gpio_sysfs_doc.png)

### 3.2 基于spidev编写APP

源码：

![image-20220323120846650](pic/69_oled_src.png)