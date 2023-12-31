# 软件工程师眼里的USB电气信号 #

参考资料：

* 《圈圈教你玩USB》
* 简书jianshu_kevin@126.com的文章
  * [USB协议（一）](https://www.jianshu.com/p/3afc1eb5bd32)
  * [USB协议(二)](https://www.jianshu.com/p/cf8e7df5ff09)
  * [USB协议(三)](https://www.jianshu.com/p/2a6e22194cd3)
* 官网：https://www.usb.org/documents
  ![image-20220721151007143](pic/02_usb_doc.png)
* 《usb_20.pdf》的《Chapter 7 Electrical》
* USB的NRZI信号格式：https://zhuanlan.zhihu.com/p/460018993
* USB2.0包Packet的组成：https://www.usbzh.com/article/detail-459.html



## 1. 学习的起点

USB 2.0协议支持3种速率：低速(Low Speed，1.5Mbps)、全速(Full Speed, 12Mbps)、高速(High Speed, 480Mbps)。

USB Hub、USB设备，也分为低速、全速、高速三种类型。一个USB设备，可能兼容低速、全速，可能兼容全速、高速，但是不会同时兼容低速、高速。



### 1.1 USB设备状态切换图

![image-20220810112912052](pic/13_state_switch.png)



### 1.2 硬件线路

下图是兼容高速模式的USB收发器电路图：

![image-20220810113117223](pic/14_transceiver_circuit.png)

USB连接涉及Hub Port和USB设备，硬件连接如下：

![image-20220810114230845](pic/15_usb_hub_connect_device.png)



## 2. 电子信号

USB连接线有4条：5V、D+、D-、GND。数据线D+、D-，只能表示4种状态。USB协议中，很巧妙地使用这两条线路实现了空闲(Idle)、开始(SOP)、传输数据(Data)、结束(EOP)等功能。

### 2.1 低速/全速信号电平

![](pic/07_ls_fs_signal.png)



### 2.2 高速信号电平

![](pic/08_hs_signal.png)

![](pic/09_hs_signal2.png)



### 2.3 设备连接与断开

#### 2.3.1 连接

Hub端口的D+、D-都有15K的下拉电阻，平时为低电平。全速设备内部的D+有1.5K的上拉电阻，低速设备内部的D-有1.5K的上拉电阻，连接到Hub后会导致Hub的D+或D-电平变化，Hub根据变化的引脚分辨接进来的是全速设备还是低速设备。

高速设备一开始也是作为全速设备被识别的。

全速设备、高速设备连接时，D+引脚的电平由低变高：

![image-20220810162143336](pic/18_fs_hs_connect.png)



低速设备连接时，D-引脚的电平由低变高：

![image-20220810162254772](pic/19_ls_connect.png)



#### 2.3.2 断开

对于低速、全速设备，接到Hub时导致D-或D+引脚变为高电平，断开设备后，D-或D+引脚变为低电平：

![image-20220810162412653](pic/20_ls_fs_disconnect.png)



对于高速设备，它先作为全速设备被识别出来，然后再被识别为高速设备。工作于高速模式时，D+的上拉电阻是断开的，所以对于工作于高速模式的USB设备，无法通过D+的引脚电平变化监测到它已经断开。

工作于高速模式的设备，D+、D-两边有45欧姆的下拉电阻，用来消除反射信号：

![image-20220810171222018](pic/21_hs_resistance.png)

当断开高速设备后，Hub发出信号，得到的反射信号无法衰减，Hub监测到这些信号后就知道高速设备已经断开，内部电路图如下：

![image-20220810171620458](pic/22_hs_disconnect.png)





### 2.4 复位

从状态切换图上看，一个USB设备连接后，它将会被供电，然后被复位。当软件出错时，我们也可以发出复位信号重新驱动设备。

那么，USB Hub端口或USB控制器端口如何发出复位信号？发出SE0信号，并维持至少10ms。

USB设备看到Reset信号后，需要准备接收"SetAddress()"请求；如果它不能回应这个请求，就是"不能识别的设备"。



### 2.5 设备速率识别

#### 2.5.1 低速/全速

Hub端口的D+、D-都有15K的下拉电阻，平时为低电平。全速设备内部的D+有1.5K的上拉电阻，低速设备内部的D-有1.5K的上拉电阻，连接到Hub后会导致Hub的D+或D-电平变化，Hub根据变化的引脚分辨接进来的是全速设备还是低速设备。

![image-20220810121433762](pic/16_ls_fs_identification.png)

#### 2.5.2 高速

高速设备必定兼容全速模式，所以高速设备内部D+也有1.5K的上拉电阻，只不过这个电阻是可以断开的：工作于高速模式时要断开它。

高速设备首先作为全速设备被识别出来，然后Hub如何确定它是否支持高速模式？

Hub端口如何监测一个新插入的USB设备能否工作于高速模式？流程如下：

* 对于低速设备，Hub端口不会监测它能否工作于高速模式。低速设备不能兼容高速模式。
* Hub端口发出SE0信号，这就是复位信号
* USB设备监测到SE0信号后，会发出"a high-speed detection handshake"信号表示自己能支持高速模式，这可以细分为一下3种情景
  * 如果USB设备原来处于"suspend"状态，它检测到SE0信号后，就发出"a high-speed detection handshake"信号
  * 如果USB设备原来处于"non-suspend"状态，并且处于全速模式，它检测到SE0信号后，就发出"a high-speed detection handshake"信号。这个情景，就是一个设备刚插到Hub端口时的情况，它一开始工作于全速模式。
  * 如果USB设备原来处于"non-suspend"状态，并且处于高速模式，它会切换回到全速模式(重新连接D+的上拉电阻)，然后发出"a high-speed detection handshake"信号



"a high-speed detection handshake"信号，就是"高速设备监测握手信号"，既然是握手信号，自然是有来有回：

* USB设备维持D+的上拉电阻，发出"Chirp K "信号，表示自己能支持高速模式
* 如果Hub没监测到"Chirp K "信号，它就知道这个设备不支持高速模式
* 如果Hub监测到"Chirp K "信号后，如果Hub能支持高速模式，就发出一系列的"Chirp K"、"Chirp J"信号，这是用来通知USB设备：Hub也能支持高速模式。发出一系列的"Chirp K"、"Chirp J"信号后，Hub继续维持SE0信号直到10ms。
* USB设备发出"Chirp K "信号后，就等待Hub回应一系列的"Chirp K"、"Chirp J"信号
  * 收到一系列的"Chirp K"、"Chirp J"信号：USB设备端口D+的上拉电阻，使能高速模式
  * 没有收到一系列的"Chirp K"、"Chirp J"信号：USB设备转入全速模式

![image](pic/17_hs_identification.png)

### 2.6 数据信号

![Packet内容](pic/23_usb_packet.png)

#### 2.6.1 低速/全速的SOP和EOP	

SOP：Start Of Packet，Hub驱动D+、D-这两条线路从Idle状态变为K状态。SOP中的K状态就是SYNC信号的第1位数据，SYNC格式为3对KJ外加2个K。

EOP：End Of Packet，由数据的发送方发出EOP，数据发送方驱动D+、D-这两条线路，先设为SE0状态并维持2位时间，再设置为J状态并维持1位时间，最后D+、D-变为高阻状态，这时由线路的上下拉电阻使得总线进入Idle状态。

![image-20220810094240442](pic/10_ls_fs_sop_eop_signal.png)



#### 2.6.2 高速的SOP

高速的EOP比较复杂，作为软件开发人员无需掌握。

高速模式中，Ide状态为：D+、D-接地。SOP格式为：从Idle状态切换为K状态。SOP中的K状态就是SYNC信号的第1位数据。

高速模式中的SYNC格式为：KJKJKJKJ KJKJKJKJ KJKJKJKJ KJKJKJKK，即15对KJ，外加2个K。



#### 2.6.3 NRZI与位填充

参考文章：[USB的NRZI信号格式](https://zhuanlan.zhihu.com/p/460018993)

NRZI：Non Return Zero Inverted Code，反向不归零编码。NRZI的编码方位为：对于数据0，波形翻转；对于数据1，波形不变。

![image-20220810100810755](pic/11_nrzi.png)

使用NRZI，发送端可以很巧妙地把"时钟频率"告诉接收端：只要传输连续的数据0即可。在下图中，低速/全速协议中"Sync Pattern"的原始数据是"00000001"，接收端从前面的7个0波形就可以算出"时钟频率"。

![image-20220810101040921](pic/12_bit_stuffing.png)



使用NRZI时，如果传输的数据总是"1"，会导致波形维持不变。如果电平长时间维持不变，比如传输100位1时，如果接收方稍有偏差，就可能认为接收到了99位1、101位1。而USB中采用了Bit-Stuffing位填充处理，即在连续发送6个1后面会插入1个0，强制翻转发送信号，从而让接收方调整频率，同步接收。而接收方在接收时只要接收到连续的6个1后，直接将后面的0删除即可恢复数据的原貌。

NRZI数据格式如上图所示。





