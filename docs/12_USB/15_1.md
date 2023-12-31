# Gadget应用实例之adb #

参考资料：

* 《Linux usb 7. Linux 配置 ADBD》https://blog.csdn.net/pwl999/article/details/121873236
* https://blog.csdn.net/kunkliu/article/details/122746762
* usb functionfs通信机制分析: https://blog.csdn.net/baidxi/article/details/123618276
* linux usb gadget functionfs的使用: https://blog.csdn.net/baidxi/article/details/122874525



## 1. ADB框架

![image-20230329145117645](pic/119_adb_frame.png)

ADB全称为"Android Debug Bridge"，Android调试桥。在纯Linux系统中也可以使用。它是client-server架构，由三部分组成：

* adbclient：我们运行的adb命令就属于adbclient，比如我们运行以下命令`adb push d:\1.txt /root`时，它就是一个adbclient，它通过adbserver把windows下的文件"d:\1.txt"推送到开发板的/root目录
* adbserver：作为一个后台程序运行运行于PC，它负责管理PC和开发板之间的通信，完成adbclient的请求
* adbd：运行于开发板的守护进程，它通过底下的Gadget跟adbserver通信

实际上，adbclient和adbserver都是同一个应用程序：比如Windows下的adb.exe，使用不同的参数来启动时就可以作为adbclient或者adbserver。我们第1次执行adb命令时，它会帮我们启动一个adb程序作为adbserver。



## 2. 体验ADB

### 2.1 在Windows安装软件

解压GIT仓库如下文件：

![image-20230329151437797](pic/120_adb_app.png)

确认里面的adb.exe所在目录，把这个目录添加进Windows的Path环境变量里。



### 2.2 在STM32MP157上实验

STM32MP157的出厂系统已经安装好了adbd，可以直接连接USB线进行测试。

比如在Windows上执行命令：

```shell
adb  devices  # 列出adb设备
adb  push  d:\1.txt  /root  # 上传文件到开发板/root目录
adb  shell   # 启动adb命令行
```



IMX6ULL的出厂系统还没安装adbd，等移植ADB时再进行实验。



## 3. functionfs

我们关注的是Gadget部分：使用legacy的方法时，我们需要在驱动程序里指定设备信息（比如设备描述符、配置描述符等等），还需要在驱动程序里实现数据的传输功能，这都在驱动程序里限定死了。

使用configfs时，我们可以灵活地指定设备信息、灵活地选择各种function。但是，还不够灵活：你必须选择某个function，这个function里已经实现实现了数据的传输功能，你无法更改。

我们能否把Gadget设备的端点暴露给用户程序？让用户程序自己操作端点来传输数据？可以！这就是functionfs。

functionfs是一种文件系统，它的使用分为两步：

* 内核态：注册functionfs
* 用户态：挂载functionfs

抓住这两点来分析代码。

### 3.1 注册functionfs

以legacy的方式来分析，只要安装g_ffs.ko驱动程序：

```shell
insmod g_ffs.ko
```

就会触发以下调用过程：

```shell
# drivers\usb\gadget\legacy\g_ffs.c
gfs_init
	usb_get_function_instance("ffs");
		try_get_usb_function_instance
			fi = fd->alloc_inst();
				# drivers\usb\gadget\function\f_fs.c
				ffs_alloc_inst
					dev = _ffs_alloc_dev();
						ret = functionfs_init();
							ret = register_filesystem(&ffs_fs_type);
```



使用configfs方式的话，需要执行以下命令：

```shell
modprobe libcomposite
mount -t configfs none /sys/kernel/config

mkdir -p /sys/kernel/config/usb_gadget/g1
mkdir  -p /sys/kernel/config/usb_gadget/g1/functions/ffs.adb
```

可以看到提示信息：

![image-20230329153126686](pic/121_ffs.png)



执行命令`cat /proc/filesystems`可以看到functionfs。



### 3.2 挂载functionfs

这时就可以挂载functionfs了，执行如下命令：

```shell
# mkdir -p /dev/usb-ffs/adb
# mount -t functionfs adb /dev/usb-ffs/adb  # 上面创建了functions/ffs.adb, 挂载时dev就要指定为adb
# ls /dev/usb-ffs/adb/
ep0
```

有了ep0断点后，用户态程序就可以通过它跟主机通信了。



### 3.3 ep0的驱动程序

ep0对应的驱动程序，分析如下：

* 挂载functionfs时，会导致一个函数被调用：

  ![image-20230329161923458](pic/124_functionfs_mount.png)

* ffs_sb_fill中，会在functionfs的根目录下创建名为ep0的文件，并给它提供file_operations结构体：
  ![image-20230329162508466](pic/125_ep0_fops.png)



## 4. ADB实现速览

源码：https://www.github.com/hadess/adbd

我们也下载放在GIT仓库了：

![image-20230329161200248](pic/122_adb_source.png)

我们关注的不是adb本身的实现，而是数据如何传输。

分析文件：adbd-master\adb\usb_linux_client.cpp

### 4.1 初始化接口描述符

![image-20230329161700570](pic/123_interface_desc.png)



### 4.2 申请更多端点

在接口描述符里，定义了多个接口描述符，这是APP提出的请求。如果Gadget设备有足够的端点，那么就会在在functionfs跟目录下创建出这些端点，比如ep1、ep2。

ADB程序的调用关系如下：

```c
init_functionfs

    // 设置功能描述符(接口描述符)
    v2_descriptor.fs_count = 3;
    v2_descriptor.hs_count = 3;
    v2_descriptor.ss_count = 5;
    v2_descriptor.os_count = 1;
    v2_descriptor.fs_descs = fs_descriptors;
    v2_descriptor.hs_descs = hs_descriptors;
    v2_descriptor.ss_descs = ss_descriptors;
    v2_descriptor.os_header = os_desc_header;
    v2_descriptor.os_desc = os_desc_compat;

	h->control = adb_open(USB_FFS_ADB_EP0, O_RDWR); // 打开端点0

	// 把接口描述符发给驱动程序
	ret = adb_write(h->control, &v2_descriptor, sizeof(v2_descriptor));

	// 发送字符串描述符, 这会触发驱动程序根据接口描述符创建更多的endpoint
	ret = adb_write(h->control, &strings, sizeof(strings));
```



上面的函数操作的都是ep0，对应的驱动程序如下：

![image-20230329164059336](pic/126_ep0_write.png)



函数ffs_epfiles_create会根据接口描述符申请更多的endpoint，并且在functionfs里创建对应的节点：

![image-20230329164221827](pic/127_create_ep.png)



## 5. 移植ADB

### 5.1 交叉编译adb

如果不想自己编译，可以使用GIT仓库里的可执行程序：

![image-20230330093712296](pic/129_adb.png)



以IMX6ULL为例，打开《嵌入式Linux应用开发完全手册V5_IMX6ULL_Pro开发板.pdf》，找到《6.5 构建IMX6ULL Pro版的根文件系统》章节，执行以下命令：

```shell
make clean
make 100ask_imx6ull_pro_ddr512m_systemV_qt5_defconfig
make menuconfig
```

配置ADB：-> Target packages  -> System tools

![image-20230329193231265](pic/128_buildroot_adb.png)

然后执行：

```shell
make android-tools-rebuild
```



期间会自动下载源码、编译。

成功后，可在如下目录查看到可执行程序adb、adbd：

```shell
/home/book/100ask_imx6ull-sdk/Buildroot_2020.02.x/output/target/usr/bin
```



把可执行程序放到开发板的/usr/bin目录。





### 5.2 脚本

IMX6ULL上使用的简化脚本：

```shell
modprobe libcomposite
mount -t configfs none /sys/kernel/config
mkdir -p /dev/usb-ffs/adb
mkdir -p /sys/kernel/config/usb_gadget/g1 
mkdir  -p /sys/kernel/config/usb_gadget/g1/functions/ffs.adb
mkdir  -p /sys/kernel/config/usb_gadget/g1/configs/b.1
ln -s  /sys/kernel/config/usb_gadget/g1/functions/ffs.adb /sys/kernel/config/usb_gadget/g1/configs/b.1
mount -t functionfs adb /dev/usb-ffs/adb
start-stop-daemon --start --oknodo --pidfile /var/run/adbd.pid --startas /usr/bin/adbd --background
sleep 1
echo ci_hdrc.0 > /sys/kernel/config/usb_gadget/g1/UDC
```



可以在/etc/init.d/目录下创建一个S99adbd文件，就可以自动使能ADB功能。这个文件在GIT仓库里：

![image-20230330093631712](pic/130_adb_sh.png)



来自STM32MP157的供参考的脚本：

```shell
#!/bin/bash -e
### BEGIN INIT INFO
# Provides:          adbd
# Required-Start:
# Required-Stop:
# Default-Start:
# Default-Stop:
# Short-Description:
# Description:       Linux ADB
### END INIT INFO

VENDOR_ID="0x1d6b"
PRODUCT_ID="0x0104"
UDC=`ls /sys/class/udc/ | awk '{print $1}'`

start() {
        mkdir -p /dev/usb-ffs/adb -m 0770

        mkdir -p /sys/kernel/config/usb_gadget/g1  -m 0770

        echo ${VENDOR_ID} > /sys/kernel/config/usb_gadget/g1//idVendor
        echo ${PRODUCT_ID} > /sys/kernel/config/usb_gadget/g1//idProduct

        mkdir  -p /sys/kernel/config/usb_gadget/g1/strings/0x409   -m 0770

        echo "0123456789ABCDEF" > /sys/kernel/config/usb_gadget/g1/strings/0x409/serialnumber
        echo "STMicroelectronics"  > /sys/kernel/config/usb_gadget/g1/strings/0x409/manufacturer
        echo "STM32MP1"  > /sys/kernel/config/usb_gadget/g1/strings/0x409/product

        mkdir  -p /sys/kernel/config/usb_gadget/g1/functions/ffs.adb
        mkdir  -p /sys/kernel/config/usb_gadget/g1/configs/b.1  -m 0770
        mkdir  -p /sys/kernel/config/usb_gadget/g1/configs/b.1/strings/0x409  -m 0770

        ln -s  /sys/kernel/config/usb_gadget/g1/functions/ffs.adb /sys/kernel/config/usb_gadget/g1/configs/b.1
        echo "adb" > /sys/kernel/config/usb_gadget/g1/configs/b.1/strings/0x409/configuration
        mount -t functionfs adb /dev/usb-ffs/adb

        start-stop-daemon --start --oknodo --pidfile /var/run/adbd.pid --startas /bin/adbd --background

        sleep 1

        echo $UDC > /sys/kernel/config/usb_gadget/g1/UDC
}

stop() {
        start-stop-daemon --stop --oknodo --pidfile /var/run/adbd.pid --retry 5
        umount /dev/usb-ffs/adb
}

restart() {
        echo $UDC > /sys/kernel/config/usb_gadget/g1/UDC
}

if [  "$UDC" != "" ]; then
        case $1 in
                start|stop|restart) "$1" ;;
        esac
fi

exit $?
```



