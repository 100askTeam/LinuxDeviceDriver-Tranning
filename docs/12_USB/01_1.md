# USB视频介绍及资料下载 #

## 1. 资料下载

GIT仓库：

```shell
https://e.coding.net/weidongshan/linux/doc_and_source_for_drivers.git
```

注意：上述链接无法用浏览器打开，必须使用GIT命令来克隆。

GIT简明教程：http://download.100ask.org/tools/Software/git/how_to_use_git.html

下载到GIT仓库后，USB资料在里面：

![image-20220721102942049](pic/01_git.png)



## 2. USB视频涉及的内容

* 必须掌握的概念
  * 硬件框架、软件框架
  * USB设备识别过程
  * 描述符等概念
* APP开发
  * libusb接口函数
  * 编写APP操作USB设备

* USB设备驱动
  * 操作USB设备的内核接口函数
  * 编写USB设备驱动程序
  * USB摄像头的编程
  * USB Audio Class 讲讲就更好了(留到ALSA部分讲)
* USB控制器驱动
  * USB设备的枚举过程
    * usb系统中的拓扑（命名）
    * usb 枚举(外接hub)，结合硬件电路图来说明
  * USB设备的读写流程
* OTG
  * type c硬件接口、micro usb硬件接口
  * OTG角色触发过程
  * OTG硬件调试经验
  * OTG使用
* Gadget
  * function和composite怎么联系起来的，可以结合zero.c那个驱动讲讲
  * Gadget的几个使用示例

