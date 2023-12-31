# OLED模块上机实验 #

参考资料：

* 内核驱动：`drivers\spi\spidev.c`

* 内核提供的测试程序：`tools\spi\spidev_fdx.c`

* 内核文档：`Documentation\spi\spidev`	

  

## 1. 连接

### 1.1 IMX6ULL

OLED模块接到IMX6ULL扩展板的SPI_A插座上：

![](pic/50_oled_on_extend_brd.png)



### 1.2 STM32MP157

OLED模块接到STM32MP157扩展板的SPI_A插座上：

![image-20220314171907903](pic/67_oled_on_stm32mp157.png)







## 2. 编译替换设备树

### 2.1 IMX6ULL

#### 2.1.1 设置工具链

```shell
export ARCH=arm
export CROSS_COMPILE=arm-buildroot-linux-gnueabihf-
 export PATH=$PATH:/home/book/100ask_imx6ull-sdk/ToolChain/arm-buildroot-linux-gnueabihf_sdk-buildroot/bin
```


#### 2.1.2 编译、替换设备树

  * 修改设备树`arch/arm/boot/dts/100ask_imx6ull-14x14.dts`
    
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



### 2.2 STM32MP157

#### 2.2.1 设置工具链

```shell
export ARCH=arm
export CROSS_COMPILE=arm-buildroot-linux-gnueabihf-
export PATH=$PATH:/home/book/100ask_stm32mp157_pro-sdk/ToolChain/arm-buildroot-linux-gnueabihf_sdk-buildroot/bin
```


#### 2.2.2 编译、替换设备树

  * 修改设备树`arch/arm/boot/dts/stm32mp157c-100ask-512d-lcd-v1.dts`
    
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



## 3. 编译spidev驱动

首先要确定内核中已经含有spidev。在内核目录下执行make menuconfig，查看是否有改驱动，如下图：

```shell
-> Device Drivers
  -> SPI support (SPI [=y]) 
    < >   User mode SPI device driver support  
```

如果`User mode SPI device driver support`前面不是`<Y>`或`<M>`，可以输入`M`表示把它编译为模块。

* 如果已经是`<Y>`，则不用再做其他事情。
* 如果你设置为`<M>`，在内核目录下执行`make modules`，把生成的`drivers/spi/spidev.ko`复制到NFS目录备用



## 4. 编译APP

```shell
arm-buildroot-linux-gnueabihf-gcc -o spi_oled spi_oled.c
```



## 5. 上机实验

如果spidev没有被编译进内核，那么先执行：

```shell
insmod spidev.ko
```



确定设备节点：

```shell
ls /dev/spidev*
```



执行测试程序：

```shell
// IMX6ULL
./spi_oled /dev/spidev0.0 116

// STM32MP157
// 在uboot里GPIOA13被配置为open drain了, 先把它设置为推挽输出
devmem 0x50000a28 32 1; devmem 0x50002004 32 0  
./spi_oled /dev/spidev0.1 13

```













