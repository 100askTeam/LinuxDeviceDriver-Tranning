# 使用Framebuffer改造OLED驱动 #

* 源码:

  ![image-20220429174003143](pic/77_src_oled_framebuffer.png)

## 1. 思路

![image-20220429173507474](pic/76_framebuffer_and_oled_gram.png)

假设OLED的每个像素使用1位数据表示:

* Linux Framebuffer中byte0对应OLED上第1行的8个像素
* OLED显存中byte0对应OLED上第1列的8个像素



为了兼容基于Framebuffer的程序，驱动程序中分配一块Framebuffer，APP直接操作Framebuffer。

驱动程序周期性地把Framebuffer中的数据搬移到OLED显存上。

怎么搬移？

发给OLED线程的byte0、1、2、3、4、5、6、7怎么构造出来？

* 它们来自Framebuffer的byte0、16、32、48、64、80、96、112
* OLED的byte0，由Framebuffer的这8个字节的bit0组合得到
* OLED的byte1，由Framebuffer的这8个字节的bit1组合得到
* OLED的byte2，由Framebuffer的这8个字节的bit2组合得到
* OLED的byte3，由Framebuffer的这8个字节的bit3组合得到
* ……





## 2. 编程

### 2.1 Framebuffer编程

分配、设置、注册fb_info结构体。

* 分配fb_info
* 设置fb_info
  * fb_var
  * fb_fix
* 注册fb_info
* 硬件操作



### 2.2 数据搬移

创建内核线程，周期性地把Framebuffer中的数据通过SPI发送给OLED。

创建内核线程:

* 参考文件`include\linux\kthread.h`

* 参考文章：https://blog.csdn.net/qq_37858386/article/details/115573565

* kthread_create：创建内核线程，线程处于"停止状态"，要运行它需要执行`wake_up_process`

* kthread_run：创建内核线程，并马上让它处于"运行状态"

* kernel_thread

  

### 2.3 调试

配置内核，把下列配置项去掉：

![image-20220125212414098](pic/78_disable_framebuffer_console.png)

