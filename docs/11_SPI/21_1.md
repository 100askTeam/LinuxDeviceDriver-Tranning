# 编写SPI_Master驱动程序_新方法 #

本节源码：
![image-20220614165137206](pic/88_spi_master_new_src.png)

参考资料：

* 内核头文件：`include\linux\spi\spi.h`
* 内核文档：`Documentation\devicetree\bindings\spi\spi-bus.txt`	
  * 内核源码：`drivers\spi\spi.c`、`drivers\spi\spi-sh.c`

## 1. SPI驱动框架

### 1.1 总体框架

![image-20220217163316229](pic/09_spi_drv_frame.png)



### 1.2 怎么编写SPI_Master驱动

#### 1.2.1 编写设备树

在设备树中，对于SPI Master，必须的属性如下：

* #address-cells：这个SPI Master下的SPI设备，需要多少个cell来表述它的片选引脚
* #size-cells：必须设置为0
* compatible：根据它找到SPI Master驱动

可选的属性如下：

* cs-gpios：SPI Master可以使用多个GPIO当做片选，可以在这个属性列出那些GPIO
* num-cs：片选引脚总数

其他属性都是驱动程序相关的，不同的SPI Master驱动程序要求的属性可能不一样。



在SPI Master对应的设备树节点下，每一个子节点都对应一个SPI设备，这个SPI设备连接在该SPI Master下面。

这些子节点中，必选的属性如下：

* compatible：根据它找到SPI Device驱动
* reg：用来表示它使用哪个片选引脚
* spi-max-frequency：必选，该SPI设备支持的最大SPI时钟

可选的属性如下：

* spi-cpol：这是一个空属性(没有值)，表示CPOL为1，即平时SPI时钟为低电平
* spi-cpha：这是一个空属性(没有值)，表示CPHA为1)，即在时钟的第2个边沿采样数据
* spi-cs-high：这是一个空属性(没有值)，表示片选引脚高电平有效
* spi-3wire：这是一个空属性(没有值)，表示使用SPI 三线模式
* spi-lsb-first：这是一个空属性(没有值)，表示使用SPI传输数据时先传输最低位(LSB)
* spi-tx-bus-width：表示有几条MOSI引脚；没有这个属性时默认只有1条MOSI引脚
* spi-rx-bus-width：表示有几条MISO引脚；没有这个属性时默认只有1条MISO引脚
* spi-rx-delay-us：单位是毫秒，表示每次读传输后要延时多久
* spi-tx-delay-us：单位是毫秒，表示每次写传输后要延时多久



#### 1.2.2 编写驱动程序

* 核心为：分配/设置/注册spi_master结构体
* 对于老方法，spi_master结构体的核心是transfer函数



## 2. 编写程序

### 2.1 数据传输流程

![](pic/85_spi_transfer_new.png)



### 2.2 写代码



