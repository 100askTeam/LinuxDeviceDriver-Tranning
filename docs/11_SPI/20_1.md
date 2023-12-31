# 使用老方法编写的SPI_Master驱动程序上机实验 #

本节源码：
![image-20220608141607466](pic/87_virtual_master_ok_imx6ull.png)

参考资料：

* 内核头文件：`include\linux\spi\spi.h`
* 内核文档：`Documentation\devicetree\bindings\spi\spi-bus.txt`	
  * 内核源码：`drivers\spi\spi.c`、`drivers\spi\spi-sh.c`

## 1. 修改设备树

### 1.1 IMX6ULL

修改`arch/arm/boot/dts/100ask_imx6ull-14x14.dts`中，如下：

```shell
        virtual_spi_master {
                compatible = "100ask,virtual_spi_master";
                status = "okay";
                cs-gpios = <&gpio4 27 GPIO_ACTIVE_LOW>;
                num-chipselects = <1>;
                #address-cells = <1>;
                #size-cells = <0>;

                virtual_spi_dev: virtual_spi_dev@0 {
                        compatible = "spidev";
                        reg = <0>;
                        spi-max-frequency = <100000>;
                };
        };

```





### 2.2 STM32MP157

修改`arch/arm/boot/dts/stm32mp157c-100ask-512d-lcd-v1.dts`中，如下：

```shell
        virtual_spi_master {
                compatible = "100ask,virtual_spi_master";
                status = "okay";
                cs-gpios = <&gpioh 6 GPIO_ACTIVE_LOW>;
                num-chipselects = <1>;
                #address-cells = <1>;
                #size-cells = <0>;

                virtual_spi_dev: virtual_spi_dev@0 {
                        compatible = "spidev";
                        reg = <0>;
                        spi-max-frequency = <100000>;
                };
        };
```



## 2. 编译替换设备树

### 2.1 IMX6ULL

#### 2.1.1 设置工具链

```shell
export ARCH=arm
export CROSS_COMPILE=arm-buildroot-linux-gnueabihf-
 export PATH=$PATH:/home/book/100ask_imx6ull-sdk/ToolChain/arm-buildroot-linux-gnueabihf_sdk-buildroot/bin
```


#### 2.1.2 编译、替换设备树

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



## 3. 编译驱动和APP



## 4. 上机实验

先执行：

```shell
insmod virtual_spi_master.ko
```



对于IMX6ULL，因为spidev没有被编译进内核，那么再执行(对于STM32MP157，无需执行这个命令)：

```shell
insmod spidev.ko
```



确定设备节点：

```shell
ls /dev/spidev*
```



假设设备节点为`/dev/spidev32765.0`，执行测试程序(它并不会真正地读写数据)：

```shell
./spi_test  /dev/spidev32765.0  500
./spi_test  /dev/spidev32765.0  600
./spi_test  /dev/spidev32765.0  1000
```



