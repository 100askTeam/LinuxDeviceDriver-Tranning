# MIPI摄像头驱动程序分析 #

参考资料：

* 内核源码: `drivers\media\usb\uvc\uvc_driver.c`
* 全志 芯片 Linux MIPI CSI摄像头接口开发指南：https://blog.csdn.net/thisway_diy/article/details/128459836
* 瑞芯微rk3568平台摄像头控制器MIPI-CSI驱动架构梳理：https://zhuanlan.zhihu.com/p/621470492
* MIPI、CSI基础：https://zhuanlan.zhihu.com/p/599531271
* 全志T7 CSIC模块：https://blog.csdn.net/suwen8100/article/details/116226366
* 全志在线文档：https://v853.docs.aw-ol.com/soft/soft_camera/
* 参考源码： https://gitee.com/juping_zheng1/v85x-mediactrl



## 1. 术语
| 术语 | 解释 |
| ---- | ---- |
|AE(Auto Exposure)|自动曝光|
|AF(Auto Focus)   |自动对焦|
|AWB(Auto White Balance)|自动白平衡|
|3A  |指自动曝光(AE)、自动对焦(AF)和自动白平衡(AWB)算法|
|Async Sub Device|在Media Controller结构下注册的V4L2异步子设备，例|Sensor、MIPI DPHY|
|Bayer Raw（或Raw Bayer）  |Bayer是相机内部的原始图片，一般后缀为.raw|.raw格式内部的存储方式有|RGGB、BGGR、GRBG等|
|CIF     |Rockchip芯片中的VIP模块，接收Sensor数据并保存到内存中，仅转存数据，无ISP功能|
|DVP(Digital Video Port)  |一种并行数据传输接口|
|Entity  |Media Controller架构下的各节点|
|Frame   |帧|
|HSYNC   |行同步信号，HSYNC有效时，接收到的信号属于同一行|
|IOMMU(Input Output Memory Management Unit)|Rockchip芯片中的IOMMU模块，用于将物理上分散的内存页映射成CIF、ISP可见的连续内存|
|IQ(Image Quality) |指为Bayer Raw Camera调试的IQ xml，用于3A tunning|
|ISP(Image Signal Processing) |图像信号处理|
|Media Controller |Linux内核中的一种媒体框架，用于拓扑结构的管理|
|MIPI-DPHY     |Rockchip芯片中符合MIPI-DPHY协议的控制器|
|MP(Main Path)|Rockchip芯片ISP驱动的一个输出节点，一般用来拍照和抓取Raw图|
|PCLK(Pixel Clock) |指Sensor输出的Pixel Clock|
|Pipeline          |Media Controller架构的各Entity之间相互连接形成的链路|
|SP(Self Patch)    |Rockchip芯片ISP驱动的一个输出节点|
|V4L2(Video4Linux2)   |指Linux内核的视频处理模块|
|VICAP(Video Capture) |视频捕获|
|VIP(Video Input Processor)|在Rockchip芯片中，曾作为CIF的别名|
|VSYNC  |场同步信号，VSYNC有效时，接收到的信号属于同一帧|



## 2. 硬件框架

### 2.1 接口

本文参考：https://zhuanlan.zhihu.com/p/599531271，作者"一口Linux"。

MIPI包含有很多协议，比如显示设备接口、摄像头设备接口、存储设备接口等等。

经常用到的有显示设备接口、摄像头设备接口。

显示设备接口又可以分为：并行接口、串行接口。并行接口又可以分为DBI、DPI。串行接口简称DSI。

摄像头设备接口常用串行接口CSI，它可分为CSI-2、CSI-3，目前常用的是CSI-2。对于CSI-2，它的物理接口分为D-PHY、C-PHY，目前常用的是D-PHY。(DSI使用的物理接口也是D-PHY)。

![image-20231207170206555](pic/49_mipi.png)

D-PHY接口引脚示例如下：

![image-20231207160308489](pic/50_csi.png)



### 2.2 硬件结构

D-PHY接口典型图例如下：

![image-20231207160959511](pic/51_d_phy.png)



对于摄像头，D-PHY接口仅仅是用来传递数据：

* 摄像头发送数据，它被称为：CSI Transmitter
* 主控接收数据，它被称为：CSI Receiver
* 主控通过I2C接口发送控制命令，它被称为：CCI Master（CCI名为Camera Control Interface）
* 摄像头接收控制命令，它被称为：CCI Slave

![image-20231207161314039](pic/52_csi_d_phy.png)



摄像头通过CSI接口，仅仅是传递数据给主控，主控还需要更多处理，如下图：

![image-20231207162159690](pic/53_camera_to_soc.png)



有哪些处理？

根据全志V853芯片资料，可以看到下图：

![image-20231207164130520](pic/54_aw_csi.png)

它的处理过程分为：

* 输入Parser：格式解析
* ISP：Image Signal Processor，即图像信号处理器，用于处理图像信号传感器输出的图像信号
* VIPP： Video Input Post Processor（视频输入后处理器），能对图片进行缩小和打水印处理。VIPP支持bayer raw data经过ISP处理后再缩小，也支持对一般的YUV格式的sensor图像直接缩小。



摄像头数据处理的完整硬件框图如下（以全志为例）：https://v853.docs.aw-ol.com/soft/soft_camera/

![image-20231207164649560](pic/55_aw_mipi_csi.png)

![image-20231207165839130](pic/56_aw_mipi_csi_2.png)

可以跟瑞芯微的对比一下：https://zhuanlan.zhihu.com/p/599531271

