# 编写USB鼠标驱动程序 #

参考资料：

* Linux内核源码：`include\linux\usb.h`

* Linux内核源码：`drivers\hid\usbhid\usbmouse.c`

* 本视频源码在GIT仓库如下位置：

  ```c
  doc_and_source_for_drivers\IMX6ULL\source\12_USB\04_usbmouse_as_key
  ```

  


## 1. 目标

使用鼠标模拟按键：左键相当于"L"、右键相当于"S"、"中键"相当于 "回车"。



## 2. 编程

### 2.1 驱动框架

![image-20221011095909341](pic/66_usb_input_frame.png)



对于GPIO按键，是直接构造、注册一个input_dev结构体，在GPIO中断函数里获得数据。

现在数据来源发生了变化，数据来自USB设备，需要做的事情是：

* 构造、注意usb_driver
* usb_driver发现能支持是设备后，它的probe函数被调用：
  * 构造、注册input_dev结构体
* 获得数据：
  * 构造、提交URB
  * 在URB的回调函数里，向Input系统上报数据



### 2.2 实现usb_driver

仿照usbmouse.c如下代码构造一个usb_driver结构体：

![image-20221011091940453](pic/64_usb_driver_example.png)

核心是：

* id_table：这个驱动能支持哪些设备
* probe函数：发现能支持的设备后，probe函数记录设备信息、注册输入设备等等



#### 2.2.1 id_table

id_table是一个usb_device_id数组，示例如下：

![image-20221011092207138](pic/65_usb_id_table.png)

usb_device_id结构体定义如下：

* match_flags：表示要比较哪些信息，可以比较设备ID、DeviceClass、InterfaceClass等等
* 根据match_flags提供其他信息：比如设备ID、DeviceClass、InterfaceClass等等
* driver_info：驱动程序可能用到的一些信息

```c
struct usb_device_id {
	/* which fields to match against? */
	__u16		match_flags;

	/* Used for product specific matches; range is inclusive */
	__u16		idVendor;
	__u16		idProduct;
	__u16		bcdDevice_lo;
	__u16		bcdDevice_hi;

	/* Used for device class matches */
	__u8		bDeviceClass;
	__u8		bDeviceSubClass;
	__u8		bDeviceProtocol;

	/* Used for interface class matches */
	__u8		bInterfaceClass;
	__u8		bInterfaceSubClass;
	__u8		bInterfaceProtocol;

	/* Used for vendor-specific interface matches */
	__u8		bInterfaceNumber;

	/* not matched against */
	kernel_ulong_t	driver_info
		__attribute__((aligned(sizeof(kernel_ulong_t))));
};
```



#### 2.2.2 probe函数

probe函数原型如下：

```c
int (*probe) (struct usb_interface *intf,
              const struct usb_device_id *id);
```

第1个参数是"struct usb_interface *"类型，表示匹配到的"USB逻辑设备"。

第2个参数是"struct usb_device_id *"类型，它是usb_driver的id_table中的某项，表示第1个参数就是跟这个usb_device_id匹配的。有必要的话，probe函数里可以从id->driver_info得到驱动相关的一些信息。

在probe函数，一般要记录intf信息，以后发起USB传输时会用到intf信息。



### 2.3 实现输入设备

核心是：分配、设置、注册一个input_device结构体。



### 2.4 实现数据传输

分配、填充、提交URB，在URB的回调函数里上报"input_event"。



## 3. 上机实验

需要重新配置内核，去掉内核自带的驱动程序。在内核目录下执行"make  menuconfig"：

```shell
Device Drivers  --->
    HID support  --->
        USB HID support  --->
            < > USB HID transport layer  // 不要选中
```



然后重新编译内核、给开发板替换内核。



测试：

```shell
# 把USB鼠标查到开发板上
# 先看看原来有哪些设备节点
ls /dev/input/event*

# 安装驱动程序
insmod usbmouse_as_key.ko

# 再看看新得到了哪个设备节点
ls /dev/input/event*

# 执行命令, 假设event4是新节点
hexdump /dev/input/event4

# 点击鼠标按键即可观察输出信息

# 第2种测试方法: 执行以下命令，按鼠标左键、右键，再按中键就有输出"ls"
cat /dev/tty0

# 第3种测试方法: 执行以下命令(注意"<"号前后没有空格)，就可以使用鼠标按键在控制台输入字符
exec 0</dev/tty0
```



