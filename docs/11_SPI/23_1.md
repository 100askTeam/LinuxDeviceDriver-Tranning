# SPI_Slave_Mode驱动程序框架 #

* 参考内核源码: `Linux-5.4\drivers\spi\spi-imx.c`
* 注意：Linux 4.9的内核未支持SPI Slave Mode
* 参考文档：《Linux_as_an_SPI_Slave_Handouts.pdf》

## 1. Master和Slave模式差别

以IMX6ULL为例，Master和Slave模式的异同如下图：

* 初始化

  * SPI寄存器有所不同
  * 引脚有所不同：Master模式下SS、CLK引脚是输出引脚，Slave模式下这些引脚是输入引脚

* 准备数据：都需要准备好要发送的数据，填充TXFIFO

* 发起传输：Master模式下要主动发起传输，等待发送完成(等待中断)

* 使能接收中断：Slave模式下无法主动发起传输，只能被动地等待，使能接收中断即可

* 中断函数里：读取RXFIFO得到数据

* 传输完成

  

![image-20220701144641150](pic/90_imx6ull_spi_master_slave.png)

## 2. SPI传输概述

### 2.1 数据组织方式

使用SPI传输时，最小的传输单位是"spi_transfer"，

对于一个设备，可以发起多个spi_transfer，

这些spi_transfer，会放入一个spi_message里。

* spi_transfer：指定tx_buf、rx_buf、len
  ![image-20220531155456156](pic/80_spi_transfer.png)

* 同一个SPI设备的spi_transfer，使用spi_message来管理：
  ![image-20220531155559288](pic/81_spi_message.png)

* 同一个SPI Master下的spi_message，放在一个队列里：
  ![image-20220531155857825](pic/82_spi_master.png)



所以，反过来，SPI传输的流程是这样的：

* 从spi_master的队列里取出每一个spi_message
  * 从spi_message的队列里取出一个spi_transfer
    * 处理spi_transfer

### 2.2 SPI控制器数据结构

参考内核文件：`include\linux\spi\spi.h`

Linux中使用spi_master结构体描述SPI控制器，有两套传输方法：

![image-20220531161952795](pic/83_spi_master.png)



## 3. SPI Slave Mode数据传输过程

![image-20220531162422353](pic/94_spi_transfer_for_slave_mode.png)



## 4. 如何编写程序

### 4.1 设备树

* SPI控制器设备树节点中，需要添加一个空属性：spi-slave  
* 要模拟哪类slave设备？需要添加`slave`子节点，这是用来指定`slave protocol`

![image-20220701152349072](pic/91_spi_slave_dts.png)



### 4.2 内核相关

* 新配置项：CONFIG_SPI_SLAVE
* 设备树的解析：增加对`spi-slave`、`slave`子节点的解析
* sysfs：新增加`/sys/devices/.../CTLR/slave`，对应`SPI Slave handlers`
* 新API
  * spi_alloc_slave( )
  * spi_slave_abort( )
  * spi_controller_is_slave( )

* 硬件设置不一样
  * master：SS、SCLK引脚是输出引脚
  * slave：SS、SCLK引脚是输入引脚
* 传输时等待函数不一样
  * master：master主动发起spi传输，它可以指定超时时间，使用函数`wait_for_completion_timeout() `进行等待
  * slave：slave只能被动等待传输，它无需指定超时时间，使用函数`wait_for_completion_interruptible()`进行等待 ，使用`.slave_abort() `来取消等待



### 4.3 简单的示例代码

#### 4.3.1 master和slave驱动示例

![image-20220701154533282](pic/92_master_slave_drv_eg.png)

#### 4.3.2 master和slave使用示例

对于设置为spi master模式的spi控制器，下面接的是一个一个spi slave设备，我们编写各类spi slave driver，通过spi master的函数读写spi slave 设备。

对于设置为spi slave模式的spi控制器，它是作为一个spi slave设备被其他单板的spi master设备来访问，它如何接收数据、如何提供数据？我们编写对应的spi slave handler。

对于master和slave，有两个重要概念：

* SPI Slave Driver：通过SPI Master控制器跟SPI Slave设备通信
* SPI Slave Handler：通过SPI Slave控制器监听远端的SPI Master

![image-20220701155549194](pic/93_spi_slave_driver_spi_slave_handler.png)

