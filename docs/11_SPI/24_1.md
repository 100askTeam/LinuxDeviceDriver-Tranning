# SPI_Slave_Mode驱动程序源码解读 #

* 参考内核源码: `Linux-5.4\drivers\spi\spi-imx.c`
* 注意：Linux 4.9的内核未支持SPI Slave Mode
* 参考文档：《Linux_as_an_SPI_Slave_Handouts.pdf》

## 1. 设备树

Linux 5.4中IMX6ULL的设备树代码里，并未支持slave模式，下图是摘自《Linux_as_an_SPI_Slave_Handouts.pdf》：

![image-20220704173545434](pic/102_dts_for_slave_mode.png)



## 2. 控制器驱动程序

在Linux 5.4中，SPI控制器的驱动程序仍然使用原理的名字：struct spi_master，但是它已经是一个宏：

```c
#define spi_master			spi_controller
```

这意味着它可以工作于master模式，也可以工作于slave模式。



### 2.1 分配一个spi_controller

![image-20220704164743728](pic/95_alloc_spi_controller.png)



### 2.2 设置spi_controller

工作于slave模式时，跟master模式最大的差别就是如下函数不一样：

* bitbang.txrx_bufs函数不一样
  * 在IMX6ULL中，这个函数为spi_imx_transfer，里面对master、slave模式分开处理
  * master模式：使用`spi_imx_pio_transfer`函数
  * slave模式：使用`spi_imx_pio_transfer_slave`函数
* 增加了bitbang.master->slave_abort函数

![image-20220704170732231](pic/97_spi_controller_set_for_slave.png)



#### 2.2.1 传输函数对比

![image-20220704171256262](pic/98_tx_rx_bufs_for_master_and_slave.png)



### 2.3 注册spi_controller

无论是master模式，还是slave模式，注册函数时一样的：

![image-20220704171538843](pic/99_register_spi_controller.png)

在spi_bitbang_start内部，会处理设备树中的子节点，创建并注册spi_device：

```c
spi_bitbang_start
    spi_register_master
    	spi_register_controller
    		of_register_spi_devices
```

![image-20220704172322899](pic/100_of_register_spi_devices.png)



对于master模式，设备树中的子节点对应真实的spi设备；

但是对于slave模式，这些子节点只是用来选择对应的spi slave handler：就是使用哪个驱动来模拟spi slave设备，比如：

![image-20220704173037387](pic/101_dts_and_slave_handler.png)



### 2.4 硬件操作

#### 2.4.1 引脚配置

![image-20220704165300094](pic/96_pin_config_for_spi_slave.png)



## 3. 设备驱动程序

### 3.1 Master模式

![image-20220217163316229](pic/09_spi_drv_frame.png)



### 3.2 Slave模式

参考代码：`Linux-5.4\drivers\spi\spi-slave-time.c`

![image-20220705162701468](pic/103_slave_handler_dts.png)