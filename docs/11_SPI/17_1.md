# OLED_Framebuffer驱动_上机实验 #

* 源码:

  ![image-20220509140847199](pic/79_src_for_framebuffer_ok.png)
  
* DMA参考文章：https://www.kernel.org/doc/html/latest/core-api/dma-api-howto.html

## 1. 硬件

### 1.1 原理图

IMX6ULL:

![image-20220316120535482](pic/47_imx6ull_oled_sch.png)

STM32MP157:

![image-20220316120427564](pic/48_stm32mp157_oled_sch.png)



原理图：

![image-20220314114847437](pic/49_oled_sch.png)



### 1.2 连接

无论是使用IMX6ULL开发板还是STM32MP157开发板，都有类似的扩展板。把OLED模块接到扩展板的SPI_A插座上，如下：

![image-20220314115446373](pic/50_oled_on_extend_brd.png)





## 2. 编写设备树

### 2.1.1 IMX6ULL

![](pic/44_imx6ull_pro_extend_spi_a.png)

DC引脚使用GPIO4_20，也需在设备树里指定。

设备树如下：arch/arm/boot/dts/100ask_imx6ull-14x14.dts

```shell
&ecspi1 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_ecspi1>;

    fsl,spi-num-chipselects = <2>;
    cs-gpios = <&gpio4 26 GPIO_ACTIVE_LOW>, <&gpio4 24 GPIO_ACTIVE_LOW>;
    status = "okay";

    oled: oled {
        compatible = "100ask,oled";
        reg = <0>;
        spi-max-frequency = <10000000>;
        dc-gpios = <&gpio4 20 GPIO_ACTIVE_HIGH>; 
    };
};
```





### 2.1.2 STM32MP157

![](pic/45_stm32mp157_pro_extend_spi_a.png)

DC引脚使用GPIOA_13，也需要在设备树里指定。

设备树如下：`arch/arm/boot/dts/stm32mp157c-100ask-512d-lcd-v1.dts`

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
                compatible = "100ask,oled";
                spi-max-frequency = <10000000>;
                reg = <1>;
                dc-gpios = <&gpioa 13 GPIO_ACTIVE_HIGH>;
        };
};
```



## 3. 编译替换设备树

### 3.1 IMX6ULL

#### 3.1.1 设置工具链

```shell
export ARCH=arm
export CROSS_COMPILE=arm-buildroot-linux-gnueabihf-
 export PATH=$PATH:/home/book/100ask_imx6ull-sdk/ToolChain/arm-buildroot-linux-gnueabihf_sdk-buildroot/bin
```


#### 3.1.2 编译、替换设备树

  * 编译设备树：
    在Ubuntu的IMX6ULL内核目录下执行如下命令,
    得到设备树文件：`arch/arm/boot/dts/100ask_imx6ull-14x14.dtb`

    ```shell
    make dtbs
    ```

  * 复制到NFS目录：

    ```shell
    $ cp arch/arm/boot/dts/100ask_imx6ull-14x14.dtb ~/nfs_rootfs/
    ```

  * 开发板上挂载NFS文件系统

    ```shell
    [root@100ask:~]#  mount -t nfs -o nolock,vers=3 192.168.1.137:/home/book/nfs_rootfs /mnt
    ```

* 更新设备树

    ```shell
    [root@100ask:~]# cp /mnt/100ask_imx6ull-14x14.dtb /boot
    [root@100ask:~]# sync
    ```

* 重启开发板



### 3.2 STM32MP157

#### 3.2.1 设置工具链

```shell
export ARCH=arm
export CROSS_COMPILE=arm-buildroot-linux-gnueabihf-
export PATH=$PATH:/home/book/100ask_stm32mp157_pro-sdk/ToolChain/arm-buildroot-linux-gnueabihf_sdk-buildroot/bin
```


#### 3.2.2 编译、替换设备树

  * 编译设备树：
    在Ubuntu的STM32MP157内核目录下执行如下命令,
    得到设备树文件：`arch/arm/boot/dts/stm32mp157c-100ask-512d-lcd-v1.dtb`

    ```shell
    make dtbs
    ```

  * 复制到NFS目录：

    ```shell
    $ cp arch/arm/boot/dts/stm32mp157c-100ask-512d-lcd-v1.dtb ~/nfs_rootfs/
    ```

  * 开发板上挂载NFS文件系统

    ```shell
    [root@100ask:~]#  mount -t nfs -o nolock,vers=3 192.168.1.137:/home/book/nfs_rootfs /mnt
    ```

* 确定设备树分区挂载在哪里

  由于版本变化，STM32MP157单板上烧录的系统可能有细微差别。
  在开发板上执行`cat /proc/mounts`后，可以得到两种结果(见下图)：

  * mmcblk2p2分区挂载在/boot目录下(下图左边)：无需特殊操作，下面把文件复制到/boot目录即可

  * mmcblk2p2挂载在/mnt目录下(下图右边)

    * 在视频里、后面文档里，都是更新/boot目录下的文件，所以要先执行以下命令重新挂载：
      * `mount  /dev/mmcblk2p2  /boot`

    ![](pic/46_boot_mount.png)

* 更新设备树

  ```shell
  [root@100ask:~]# cp /mnt/stm32mp157c-100ask-512d-lcd-v1.dtb /boot/
  [root@100ask:~]# sync
  ```

* 重启开发板



## 4. 编译OLED驱动

```shell
cd 10_oled_framebuffer_ok
make
```



## 5. 编译APP

```shell
cd  10_oled_framebuffer_ok/03_freetype_show_font_angle
make
```





## 6. 上机实验

```shell
1. 安装驱动程序, 确定新出现哪个设备节点
ls /dev/fb*
insmod oled_drv.ko
ls /dev/fb*

2. 在IMX6ULL上运行测试程序
./freetype_show_font_angle  /dev/fb2 ./simsun.ttc 0 20

2. 在STM32MP157上运行测试程序
./freetype_show_font_angle  /dev/fb1 ./simsun.ttc 0 20
```

