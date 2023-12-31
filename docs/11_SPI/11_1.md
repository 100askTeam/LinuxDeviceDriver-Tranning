# 编写SPI设备驱动程序 #

参考资料：

* 内核头文件：`include\linux\spi\spi.h`

* 内核文档：`Documentation\spi\spidev`	

  


## 1. SPI驱动程序框架

![image-20220217163316229](pic/09_spi_drv_frame.png)



## 2. 怎么编写SPI设备驱动程序

### 2.1 编写设备树

* 查看原理图，确定这个设备链接在哪个SPI控制器下

* 在设备树里，找到SPI控制器的节点

* 在这个节点下，创建子节点，用来表示SPI设备

* 示例如下：

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
          spi-max-frequency = <10000000>;
      };
  };
  ```
  
  



### 2.2 注册spi_driver

SPI设备的设备树节点，会被转换为一个spi_device结构体。

我们需要编写一个spi_driver来支持它。

示例如下：

```c
static const struct of_device_id dac_of_match[] = {
	{.compatible = "100ask,dac"},
	{}
};

static struct spi_driver dac_driver = {
	.driver = {
		.name	= "dac",
		.of_match_table = dac_of_match,
	},
	.probe		= dac_probe,
	.remove		= dac_remove,
	//.id_table	= dac_spi_ids,
};
```





### 2.3 怎么发起SPI传输

#### 2.3.1 接口函数

接口函数都在这个内核文件里：`include\linux\spi\spi.h`

* 简易函数

  ```c
  /**
   * SPI同步写
   * @spi: 写哪个设备
   * @buf: 数据buffer
   * @len: 长度
   * 这个函数可以休眠
   *
   * 返回值: 0-成功, 负数-失败码
   */
  static inline int
  spi_write(struct spi_device *spi, const void *buf, size_t len);
  
  /**
   * SPI同步读
   * @spi: 读哪个设备
   * @buf: 数据buffer
   * @len: 长度
   * 这个函数可以休眠
   *
   * 返回值: 0-成功, 负数-失败码
   */
  static inline int
  spi_read(struct spi_device *spi, void *buf, size_t len);
  
  
  /**
   * spi_write_then_read : 先写再读, 这是一个同步函数
   * @spi: 读写哪个设备
   * @txbuf: 发送buffer
   * @n_tx: 发送多少字节
   * @rxbuf: 接收buffer
   * @n_rx: 接收多少字节
   * 这个函数可以休眠
   * 
   * 这个函数执行的是半双工的操作: 先发送txbuf中的数据，在读数据，读到的数据存入rxbuf
   *
   * 这个函数用来传输少量数据(建议不要操作32字节), 它的效率不高
   * 如果想进行高效的SPI传输，请使用spi_{async,sync}(这些函数使用DMA buffer)
   *
   * 返回值: 0-成功, 负数-失败码
   */
  extern int spi_write_then_read(struct spi_device *spi,
  		const void *txbuf, unsigned n_tx,
  		void *rxbuf, unsigned n_rx);
  
  /**
   * spi_w8r8 - 同步函数，先写8位数据，再读8位数据
   * @spi: 读写哪个设备
   * @cmd: 要写的数据
   * 这个函数可以休眠
   *
   *
   * 返回值: 成功的话返回一个8位数据(unsigned), 负数表示失败码
   */
  static inline ssize_t spi_w8r8(struct spi_device *spi, u8 cmd);
  
  /**
   * spi_w8r16 - 同步函数，先写8位数据，再读16位数据
   * @spi: 读写哪个设备
   * @cmd: 要写的数据
   * 这个函数可以休眠
   *
   * 读到的16位数据: 
   *     低地址对应读到的第1个字节(MSB)，高地址对应读到的第2个字节(LSB)
   *     这是一个big-endian的数据
   *
   * 返回值: 成功的话返回一个16位数据(unsigned), 负数表示失败码
   */
  static inline ssize_t spi_w8r16(struct spi_device *spi, u8 cmd);
  
  /**
   * spi_w8r16be - 同步函数，先写8位数据，再读16位数据，
   *               读到的16位数据被当做big-endian，然后转换为CPU使用的字节序
   * @spi: 读写哪个设备
   * @cmd: 要写的数据
   * 这个函数可以休眠
   *
   * 这个函数跟spi_w8r16类似，差别在于它读到16位数据后，会把它转换为"native endianness"
   *
   * 返回值: 成功的话返回一个16位数据(unsigned, 被转换为本地字节序), 负数表示失败码
   */
  static inline ssize_t spi_w8r16be(struct spi_device *spi, u8 cmd);
  ```

* 复杂的函数

  ```c
  /**
   * spi_async - 异步SPI传输函数，简单地说就是这个函数即刻返回，它返回后SPI传输不一定已经完成
   * @spi: 读写哪个设备
   * @message: 用来描述数据传输，里面含有完成时的回调函数(completion callback)
   * 上下文: 任意上下文都可以使用，中断中也可以使用
   *
   * 这个函数不会休眠，它可以在中断上下文使用(无法休眠的上下文)，也可以在任务上下文使用(可以休眠的上下文) 
   *
   * 完成SPI传输后，回调函数被调用，它是在"无法休眠的上下文"中被调用的，所以回调函数里不能有休眠操作。
   * 在回调函数被调用前message->statuss是未定义的值，没有意义。
   * 当回调函数被调用时，就可以根据message->status判断结果: 0-成功,负数表示失败码
   * 当回调函数执行完后，驱动程序要认为message等结构体已经被释放，不能再使用它们。
 *
   * 在传输过程中一旦发生错误，整个message传输都会中止，对spi设备的片选被取消。
   *
   * 返回值: 0-成功(只是表示启动的异步传输，并不表示已经传输成功), 负数-失败码
   */
  extern int spi_async(struct spi_device *spi, struct spi_message *message);
  
  /**
   * spi_sync - 同步的、阻塞的SPI传输函数，简单地说就是这个函数返回时，SPI传输要么成功要么失败
   * @spi: 读写哪个设备
   * @message: 用来描述数据传输，里面含有完成时的回调函数(completion callback)
   * 上下文: 能休眠的上下文才可以使用这个函数
   *
   * 这个函数的message参数中，使用的buffer是DMA buffer
   *
   * 返回值: 0-成功, 负数-失败码
   */
  extern int spi_sync(struct spi_device *spi, struct spi_message *message);
  
  
  /**
   * spi_sync_transfer - 同步的SPI传输函数
   * @spi: 读写哪个设备
   * @xfers: spi_transfers数组，用来描述传输
   * @num_xfers: 数组项个数
   * 上下文: 能休眠的上下文才可以使用这个函数
   *
   * 返回值: 0-成功, 负数-失败码
   */
  static inline int
  spi_sync_transfer(struct spi_device *spi, struct spi_transfer *xfers,
  	unsigned int num_xfers);
  ```
  
  

#### 2.3.2 函数解析

在SPI子系统中，用spi_transfer结构体描述一个传输，用spi_message管理过个传输。

SPI传输时，发出N个字节，就可以同时得到N个字节。

* 即使只想读N个字节，也必须发出N个字节：可以发出0xff
* 即使只想发出N个字节，也会读到N个字节：可以忽略读到的数据。



spi_transfer结构体如下图所示：

* tx_buf：不是NULL的话，要发送的数据保存在里面
* rx_buf：不是NULL的话，表示读到的数据不要丢弃，保存进rx_buf里

![image-20220330162208146](pic/70_spi_transfer.png)



可以构造多个spi_transfer结构体，把它们放入一个spi_message里面。

spi_message结构体如下图所示：

![image-20220330162650541](pic/71_spi_message.png)



SPI传输示例：

![image-20220330163124260](pic/72_spidev_sync_write.png)



