# SPI_OLED模块操作方法 #

参考资料：

* OLEDC芯片手册：`SPEC UG-2864TMBEG01 .pdf`、`SSD1306-Revision 1.1 (Charge Pump).pdf`

  


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





## 2. OLED操作原理

原理图简化如下：

![image-20220314115726437](pic/51_oled_simple_sch.png)

要操作OLED，只需使用SPI接口发送数据，并不需要使用SPI接口读取数据。除此之外，还需要控制D/C引脚：

* 当DC引脚是低电平时，是命令：比如复位、打开显示、设置地址
* 当DC引脚是高电平时，是数据：写入要显示的数据


### 2.1 显存和像素

OLED上有128*64个像素，每个像素只有2种状态：亮、灭。

![image-20220314130622659](pic/54_lcd_pixels.png)

怎么控制屏幕上每个像素的状态？OLED内部有显存GDDRAM(Graphic Display Data RAM)：

![image-20220314130704746](pic/55_gddram.png)



显存中，每位对应一个像素，入下图所示：

* byte0的8位数据对应屏幕上角左侧、竖向排列的8个像素，即COL0的像素，bit0对应第0行，bit1对应第1行，……
* byte0对应COL1那列第0~第7行的8个像素
* ……
* byte127对应COL127那列第0~第7行的8个像素
* byte128对应COL0那列第0~第7行的8个像素

![image-20220314133340423](pic/57_gddram_pixels.png)

### 2.2 显存寻址模式

显存被分为8页、127列，要写某个字节时，需要先指定地址，然后写入1字节的数据

* 哪页(Page)？
* 哪列(Col)？
* 写入1字节数据

OLED有三种寻址模式：

* 页地址模式(Page addressing mode)：每写入1个字节，行地址不变，列地址增1，列地址达到127后会从0开始
  ![image-20220314135305565](pic/58_page_address.png)

* 水平地址模式(Horizontal  addressing mode)：

  * 每写入1个字节，行地址不变，列地址增1
  * 列地址达到127后从0开始，行地址指向下一页
  * 列地址达到127、行地址达到7时，列地址和行地址都被复位为0，指向左上角

  ![image-20220314135622413](pic/59_horizontal_address.png)

* 垂直地址模式(Vertical addressing mode)：

  * 每写入1个字节，行地址增1，列地址不变

  * 行地址达到7后从0开始，列地址指向下一列

  * 列地址达到127、行地址达到7时，列地址和行地址都被复位为0，指向左上角

    ![image-20220314135901062](pic/60_vertical_address.png)

### 2.3 具体操作

#### 2.3.1 初始化

对于OLED的初始化，在参考手册`SPEC UG-2864TMBEG01 .pdf`中列出的流程图：

![image-20220314123149156](pic/52_oled_pwr_up_seq.png)



#### 2.3.2 设置地址

* 设置地址模式
* 设置page
* 设置col

如下图：

* 设置地址模式，比如设置为页地址模式时，先写命令0x22，再写命令0x02。页地址模式是默认的，可以不设置。
  ![image-20220314152208486](pic/62_set_page_addressing_mode.png)
* 设置页地址：有0~7页，想设置哪一页(n)就发出命令：0xB0 | n
  ![image-20220314152336821](pic/63_set_page.png)

* 设置列地址：列地址范围是0~127，需要使用2个命令来设置列地址
  ![image-20220314152548116](pic/64_set_col_address.png)



#### 2.3.4 写入数据

让DC引脚为高，发起SPI写操作即可。



## 3. 在OLED上显示文字



