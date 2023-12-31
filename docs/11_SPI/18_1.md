# SPI_Master驱动程序框架 #

* 参考内核源码: `drivers\spi\spi.c`

## 1. SPI传输概述

### 1.1 数据组织方式

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

### 1.2 SPI控制器数据结构

参考内核文件：`include\linux\spi\spi.h`

Linux中使用spi_master结构体描述SPI控制器，有两套传输方法：

![image-20220531161952795](pic/83_spi_master.png)



## 2. SPI传输函数的两种方法

### 2.1 老方法

![image-20220531162353326](pic/84_spi_transfer_old.png)



### 2.2 新方法

![image-20220531162422353](pic/85_spi_transfer_new.png)

