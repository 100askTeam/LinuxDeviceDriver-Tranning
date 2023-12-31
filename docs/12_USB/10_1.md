# OTG硬件检测电路 #




## 1. OTG接口与转换器

OTG是"On The Go"的英文缩写，字面上可以理解为“安上即可用”。USB传输是主从结构，一切USB传输都有Host发起。比如在开发板上可以插入U盘，这时开发板作为USB Host。但是开发板要跟PC通信，开发板就要作为USB Device。开发板要作为USB Host、USB Device两种角色，可以使用OTG插口：它可以根据硬件电路自动识别自己的角色，切换为USB Host或USB Deivce。

OTG插口有多种形态，常用的有Micro USB、Type C，如下：

![image-20221108105715423](pic/67_otg_interface.png)

### 1.1 Micro USB

对于Micro USB插座，它有5条引脚：

![image-20221108110022192](pic/68_microusb_pin.png)

引脚作用如下表所示：

| 引脚名 | 作用                                                       |
| ------ | ---------------------------------------------------------- |
| VBUS   | 作为Host时，对外供电<br />作为Device时，接收外部输入的电源 |
| DM     | 数据信号                                                   |
| DP     | 数据信号                                                   |
| ID     | 分辨自己角色的引脚：<br />1：作为Device<br />0：作为Host   |
| GND    | 地线                                                       |



开发板作为USB Device时跟PC上的USB相连，PC的USB接口只有VBUS、DM、DP、GND，所以开发板的ID引脚跟PC的USB口并无连接，它被板子上的上拉电阻拉高。

开发板作为USB Host时，需要接入一个"OTG转换器"，如下图黑色的转换器：

![image-20221108142754022](pic/69_otg_connect.png)

OTG转换器的内部电路很简单(参考：https://www.lulian.cn/news/otg_gongneng_jiexi-cn.html)：

![image-20221108143637517](pic/70_otg_sch.png)

这个转换器插入开发板的OTG口之后，OTG口上的ID引脚就被拉低，软件转换为USB Host。



### 1.2 Type C

Type C插座里面有两组完全一样的信号，Type C数据线无论正插、反插，都可以使用：

![image-20221108144741285](pic/71_typec_signal.png)



Type C插座有如下信号(参考：https://blog.csdn.net/qq_37659014/article/details/124479125)，在USB2.0协议里我们只关心红框里的信号：

![image-20221108145705284](pic/72_typec_pins.png)



开发板作为USB Device时跟PC上的USB相连，PC的USB接口只有VBUS、DM、DP、GND，所以开发板的CC1、CC2引脚跟PC的USB口并无连接，它被板子上的上拉电阻拉高。

开发板作为USB Host时，需要接入一个"OTG转换器"，如下图黑色的转换器：

![image-20221108150227540](pic/73_otg_convertor_typec.png)

如果不考虑兼容USB 3.0协议，上述转换器的电路图很简单，把Type C插头里面的CC引脚连接5.1K欧姆电阻到GND即可。如下图所示(参考：https://www.elecfans.com/connector/20180309645002_a.html)：

![image-20221108150801086](pic/74_otg_convertor_res.png)



## 2. OTG接口电路

开发板上的OTG接口需要实现两个功能：

* 检测ID引脚(使用Type C接口的话是CC1、CC2引脚)，引入主控芯片：软件根据它设置USB控制器的角色(Host或Device)
* 根据ID引脚(或者CC1、CC2)决定VBUS是否输出电源：硬件电路自动实现



### 2.1 Micro USB

![image-20221108151831444](pic/75_micro_usb_otg_sch.png)



### 2.2 Type C

如果不考虑兼容USB 3.0协议，可以使用如下精简电路：CC1、CC2作为ID引脚。

![image-20221108152553966](pic/76_type_org_sch1.png)



如果要兼容USB 3.0协议，则需要加入专用的芯片：

![image-20221108152818579](pic/77_typec_org_sch2.png)

