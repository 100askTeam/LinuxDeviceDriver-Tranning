# 编写SPI_DAC模块驱动程序 #

参考资料：

* DAC芯片手册：`TLC5615.pdf`




## 1. 要做什么事情

![](pic/09_spi_drv_frame.png)



* 查看原理图，编写设备树
* 编写驱动程序，注册一个spidrv
* 编写测试程序




## 2. 硬件

### 2.1 原理图

IMX6ULL:

![image-20220309150927785](pic/33_imx6ull_dac.png)

STM32MP157:

![image-20220309151025637](pic/34_stm32mp157_dac.png)



原理图：

![image-20220309151636533](pic/35_dac_sch.png)



### 2.2 连接

#### 2.2.1 IMX6ULL

DAC模块接到IMX6ULL扩展板的SPI_A插座上：

![image-20220309164031109](pic/40_dac_on_imx6ull.png)



#### 2.2.2 STM32MP157



## 3. 编写设备树

确认SPI时钟最大频率：

![image-20220309163435541](pic/39_dac_time_param.png)

```shell
T = 25 + 25 = 50ns
F = 20000000 = 20MHz
```



设备树如下：

```shell
    dac: dac {
        compatible = "100ask,dac";
        reg = <0>;
        spi-max-frequency = <20000000>;
    };
```



### 3.1 IMX6ULL

![image-20220311101017666](pic/44_imx6ull_pro_extend_spi_a.png)

DAC模块接在这个插座上，那么要在设备树里spi1的节点下创建子节点。

代码在`arch/arm/boot/dts/100ask_imx6ull-14x14.dtb`中，如下：

```shell
&ecspi1 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_ecspi1>;

    fsl,spi-num-chipselects = <2>;
    cs-gpios = <&gpio4 26 GPIO_ACTIVE_LOW>, <&gpio4 24 GPIO_ACTIVE_LOW>;
    status = "okay";

    dac: dac {
        compatible = "100ask,dac";
        reg = <0>;
        spi-max-frequency = <20000000>;
    };
};
```





### 3.2 STM32MP157

![image-20220311101127305](pic/45_stm32mp157_pro_extend_spi_a.png)

DAC模块接在这个插座上，那么要在设备树里spi5的节点下创建子节点。

代码在`arch/arm/boot/dts/stm32mp157c-100ask-512d-lcd-v1.dts`中，如下：

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
        dac_test: dac_test@1{
                compatible = "100ask,dac";
                spi-max-frequency = <20000000>;
                reg = <1>;
        };
};
```





## 4. 编写驱动程序

以前我们基于spidev编写过DAC的应用程序，可以参考它：

![image-20220310120532411](pic/41_dac_app_use_spidev.png)





## 5. 编写测试程序

