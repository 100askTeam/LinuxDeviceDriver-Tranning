# USB描述符 #

参考资料：

* 《圈圈教你玩USB》
* 简书jianshu_kevin@126.com的文章
  * [USB协议（一）](https://www.jianshu.com/p/3afc1eb5bd32)
  * [USB协议(二)](https://www.jianshu.com/p/cf8e7df5ff09)
  * [USB协议(三)](https://www.jianshu.com/p/2a6e22194cd3)
* 官网：https://www.usb.org/documents
  ![image-20220721151007143](pic/02_usb_doc.png)
* 《usb_20.pdf》的《Chapter 9  USB Device Framework》

## 1. USB设备状态切换图

![image-20220810112912052](pic/13_state_switch.png)

![image-20220826100617354](pic/40_visible_device_states.png)



## 2. 标准设备请求

### 2.1 SETUP事务的数据格式

Host使用控制传输来识别设备、设置设备地址、启动设备的某些特性，对于控制传输，它首先发出"setup事务"，如下：

![](pic/37_control_transfer.png)

在"setup事务"中，

* SETUP令牌包：用来通知设备，"要开始传输了"
* DATA0数据包：它含有固定的格式，用来告诉设备"是读还是写"、"读什么"、"写什么"

Hos通过DATA0数据包发送8字节数据给设备，它的格式如下图所示：

![image-20220826110052602](pic/45_format_of_setup_data.png)



### 2.2 标准设备请求

控制传输的建立事务中，可以使用下列格式的数据：

![image-20220826151256669](pic/50_standard_device_requests.png)

上表中各个"宏"取值如下：

![image-20220826151641559](pic/51_macro_value.png)



### 2.2 设备/配置/接口/端点

在SETUP事务的数据里，表示了要访问的是什么：Device？Interface？Endpoint？

对于一个USB设备，它可以多种配置(Configuration)。比如4G上网卡就有2种配置：U盘、上网卡。第1次把4G上网卡插入电脑时，它是一个U盘，可以按照里面的程序。装好程序后，把它再次插入电脑，它就是一个上网卡。驱动程序可以选择让它工作于哪种配置，同一时间只能有一种配置。大多数的USB设备只有一种配置。

一个配置下，可以有多个接口(Interface)，接口等同于功能(Function)。比如USB耳机有两个接口(功能)：声音收发、按键控制。

一个接口，可能有多个设置(Setting)，比如默认设置下它使用较低的带宽，可以选择其他设置以使用更高带宽。

一个接口，由一个或多个端点(Endpoint)组成。端点0属于整个设备的，端点0是双向的。接口还可以有其他端点，这些端点是单向的，要么是批量(Bulk)端点、要么是中断(Interrupt)端点、要么是同步(Isochronous)端点。





## 3. 描述符

怎么描述设备、配置、接口、端点？使用描述符(Descriptors)，有设备描述符、配置描述符、接口描述符、端点描述符。所谓描述符，就是一些格式化的数据，用来描述信息。
一个USB设备，

* 只有一个设备描述符：用来表示设备的ID、它有多少个配置、它的端点0一次最大能传输多少字节数据
* 可能有多个配置描述符：用来表示它有多少个接口、供电方式、最大电流
* 一个配置描述符下面，可能有多个接口描述符：用来表示它是哪类接口、有几个设置(Setting)、有几个端点
* 一个接口描述符符下面，可能有多个端点描述符：用来表示端点号、方向(IN/OUT)、类型(批量/中断/同步)



还有一些字符串描述符(String descriptors)，它用可读的文字来描述设备，是可选的。

![](pic/53_usb_descriptor.png)



### 3.1 设备描述符

![image-20220826145201102](pic/46_device_descriptor.png)



### 3.2 配置描述符

![image-20220826145756502](pic/47_configration_descriptor.png)



### 3.3 接口描述符

![image-20220826150129657](pic/48_interface_descriptor.png)



### 3.4 端点描述符

![image-20220826150607222](pic/49_endpoint_descriptor.png)



### 3.5 示例

在Ubuntu中可以执行`lsusb -v`查看USB设备的描述符信息：

```shell
book@100ask:~$ sudo lsusb -v
[sudo] password for book:

Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               2.00
  bDeviceClass            9 Hub
  bDeviceSubClass         0 Unused
  bDeviceProtocol         0 Full speed (or root) hub
  bMaxPacketSize0        64
  idVendor           0x1d6b Linux Foundation
  idProduct          0x0002 2.0 root hub
  bcdDevice            5.04
  iManufacturer           3 Linux 5.4.0-124-generic ehci_hcd
  iProduct                2 EHCI Host Controller
  iSerial                 1 0000:02:03.0
  bNumConfigurations      1
  Configuration Descriptor:
    bLength                 9
    bDescriptorType         2
    wTotalLength           25
    bNumInterfaces          1
    bConfigurationValue     1
    iConfiguration          0
    bmAttributes         0xe0
      Self Powered
      Remote Wakeup
    MaxPower                0mA
    Interface Descriptor:
      bLength                 9
      bDescriptorType         4
      bInterfaceNumber        0
      bAlternateSetting       0
      bNumEndpoints           1
      bInterfaceClass         9 Hub
      bInterfaceSubClass      0 Unused
      bInterfaceProtocol      0 Full speed (or root) hub
      iInterface              0
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x81  EP 1 IN
        bmAttributes            3
          Transfer Type            Interrupt
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0004  1x 4 bytes
        bInterval              12
Hub Descriptor:
  bLength               9
  bDescriptorType      41
  nNbrPorts             6
  wHubCharacteristic 0x000a
    No power switching (usb 1.0)
    Per-port overcurrent protection
  bPwrOn2PwrGood       10 * 2 milli seconds
  bHubContrCurrent      0 milli Ampere
  DeviceRemovable    0x00
  PortPwrCtrlMask    0xff
 Hub Port Status:
   Port 1: 0000.0100 power
   Port 2: 0000.0100 power
   Port 3: 0000.0100 power
   Port 4: 0000.0100 power
   Port 5: 0000.0100 power
   Port 6: 0000.0100 power
Device Status:     0x0001
  Self Powered

Bus 002 Device 003: ID 0e0f:0002 VMware, Inc. Virtual USB Hub
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               1.10
  bDeviceClass            9 Hub
  bDeviceSubClass         0 Unused
  bDeviceProtocol         0 Full speed (or root) hub
  bMaxPacketSize0         8
  idVendor           0x0e0f VMware, Inc.
  idProduct          0x0002 Virtual USB Hub
  bcdDevice            1.00
  iManufacturer           1 VMware, Inc.
  iProduct                2 VMware Virtual USB Hub
  iSerial                 0
  bNumConfigurations      1
  Configuration Descriptor:
    bLength                 9
    bDescriptorType         2
    wTotalLength           25
    bNumInterfaces          1
    bConfigurationValue     1
    iConfiguration          1 VMware, Inc.
    bmAttributes         0xe0
      Self Powered
      Remote Wakeup
    MaxPower                0mA
    Interface Descriptor:
      bLength                 9
      bDescriptorType         4
      bInterfaceNumber        0
      bAlternateSetting       0
      bNumEndpoints           1
      bInterfaceClass         9 Hub
      bInterfaceSubClass      0 Unused
      bInterfaceProtocol      0 Full speed (or root) hub
      iInterface              1 VMware, Inc.
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x81  EP 1 IN
        bmAttributes            3
          Transfer Type            Interrupt
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0001  1x 1 bytes
        bInterval             255
Hub Descriptor:
  bLength               9
  bDescriptorType      41
  nNbrPorts             7
  wHubCharacteristic 0x0009
    Per-port power switching
    Per-port overcurrent protection
  bPwrOn2PwrGood       50 * 2 milli seconds
  bHubContrCurrent    100 milli Ampere
  DeviceRemovable    0x00
  PortPwrCtrlMask    0xfe
 Hub Port Status:
   Port 1: 0000.0100 power
   Port 2: 0000.0100 power
   Port 3: 0000.0100 power
   Port 4: 0000.0100 power
   Port 5: 0000.0100 power
   Port 6: 0000.0100 power
   Port 7: 0000.0100 power
Device Status:     0x2909
  Self Powered

Bus 002 Device 002: ID 0e0f:0003 VMware, Inc. Virtual Mouse
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               1.10
  bDeviceClass            0 (Defined at Interface level)
  bDeviceSubClass         0
  bDeviceProtocol         0
  bMaxPacketSize0         8
  idVendor           0x0e0f VMware, Inc.
  idProduct          0x0003 Virtual Mouse
  bcdDevice            1.03
  iManufacturer           1 VMware
  iProduct                2 VMware Virtual USB Mouse
  iSerial                 0
  bNumConfigurations      1
  Configuration Descriptor:
    bLength                 9
    bDescriptorType         2
    wTotalLength           34
    bNumInterfaces          1
    bConfigurationValue     1
    iConfiguration          1 VMware
    bmAttributes         0xc0
      Self Powered
    MaxPower                0mA
    Interface Descriptor:
      bLength                 9
      bDescriptorType         4
      bInterfaceNumber        0
      bAlternateSetting       0
      bNumEndpoints           1
      bInterfaceClass         3 Human Interface Device
      bInterfaceSubClass      1 Boot Interface Subclass
      bInterfaceProtocol      2 Mouse
      iInterface              1 VMware
        HID Device Descriptor:
          bLength                 9
          bDescriptorType        33
          bcdHID               1.10
          bCountryCode            0 Not supported
          bNumDescriptors         1
          bDescriptorType        34 Report
          wDescriptorLength      46
         Report Descriptors:
           ** UNAVAILABLE **
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x81  EP 1 IN
        bmAttributes            3
          Transfer Type            Interrupt
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0008  1x 8 bytes
        bInterval               1
Device Status:     0x0001
  Self Powered

Bus 002 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               1.10
  bDeviceClass            9 Hub
  bDeviceSubClass         0 Unused
  bDeviceProtocol         0 Full speed (or root) hub
  bMaxPacketSize0        64
  idVendor           0x1d6b Linux Foundation
  idProduct          0x0001 1.1 root hub
  bcdDevice            5.04
  iManufacturer           3 Linux 5.4.0-124-generic uhci_hcd
  iProduct                2 UHCI Host Controller
  iSerial                 1 0000:02:00.0
  bNumConfigurations      1
  Configuration Descriptor:
    bLength                 9
    bDescriptorType         2
    wTotalLength           25
    bNumInterfaces          1
    bConfigurationValue     1
    iConfiguration          0
    bmAttributes         0xe0
      Self Powered
      Remote Wakeup
    MaxPower                0mA
    Interface Descriptor:
      bLength                 9
      bDescriptorType         4
      bInterfaceNumber        0
      bAlternateSetting       0
      bNumEndpoints           1
      bInterfaceClass         9 Hub
      bInterfaceSubClass      0 Unused
      bInterfaceProtocol      0 Full speed (or root) hub
      iInterface              0
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x81  EP 1 IN
        bmAttributes            3
          Transfer Type            Interrupt
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0002  1x 2 bytes
        bInterval             255
Hub Descriptor:
  bLength               9
  bDescriptorType      41
  nNbrPorts             2
  wHubCharacteristic 0x000a
    No power switching (usb 1.0)
    Per-port overcurrent protection
  bPwrOn2PwrGood        1 * 2 milli seconds
  bHubContrCurrent      0 milli Ampere
  DeviceRemovable    0x00
  PortPwrCtrlMask    0xff
 Hub Port Status:
   Port 1: 0000.0103 power enable connect
   Port 2: 0000.0107 power suspend enable connect
Device Status:     0x0001
  Self Powered
```





## 4. 设备枚举过程示例

使用"usbprotocolsuite"打开，可以看到设备的枚举过程：

* 使用控制传输，读取设备信息(设备描述符)：第一次读取时，它只需要得到8字节数据，因为第8个数据表示端点0能传输的最大数据长度。
  ![image-20220826102651368](pic/41_get_descriptor_example.png)
* Host分配地址给设备，然后把新地址发给设备：
  ![image-20220826103029471](pic/42_set_address_example.png)
* 使用新地址，重新读取设备描述符，设备描述符长度是18：
  ![image-20220826103218424](pic/43_get_descriptor_example2.png)
* 读取配置描述符：它传入的长度是255，想一次性把当前配置描述符、它下面的接口描述符、端点描述符全部读出来
  ![image-20220826103315728](pic/44_get_config_descriptor_example2.png)
* 读取字符描述符

![image-20220826163411731](pic/52_get_string_descriptor.png)



