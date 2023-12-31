# configfs的使用与内部机制 #

参考资料：

* 内核文档：
  * Documentation\filesystems\configfs\configfs.txt
  * Documentation\usb\gadget_configfs.txt

## 1. 体验

### 1.1 使用

所有命令都是在开发板上执行。

* 挂载configfs文件系统

  ```shell
  # modprobe libcomposite
  # mount -t configfs none /sys/kernel/config
  # ls /sys/kernel/config/
  usb_gadget
  
  #ls /sys/kernel/config/usb_gadget  // 一开始它是空目录
  ```

* 创建目录

  ```shell
  # cd /sys/kernel/config/usb_gadget
  # mkdir test_serial
  # ls test_serial/ -l
  total 0
  -rw-r--r--    1 root     root          4096 Jan  1 03:20 UDC
  -rw-r--r--    1 root     root          4096 Jan  1 03:20 bDeviceClass
  -rw-r--r--    1 root     root          4096 Jan  1 03:20 bDeviceProtocol
  -rw-r--r--    1 root     root          4096 Jan  1 03:20 bDeviceSubClass
  -rw-r--r--    1 root     root          4096 Jan  1 03:20 bMaxPacketSize0
  -rw-r--r--    1 root     root          4096 Jan  1 03:20 bcdDevice
  -rw-r--r--    1 root     root          4096 Jan  1 03:20 bcdUSB
  drwxr-xr-x    2 root     root             0 Jan  1 01:49 configs
  drwxr-xr-x    2 root     root             0 Jan  1 01:49 functions
  -rw-r--r--    1 root     root          4096 Jan  1 03:20 idProduct
  -rw-r--r--    1 root     root          4096 Jan  1 03:20 idVendor
  drwxr-xr-x    2 root     root             0 Jan  1 01:49 os_desc
  drwxr-xr-x    2 root     root             0 Jan  1 01:49 strings
  ```

  创建目录后，里面就自动生成了很多文件、目录，比如：

  * idVendor：表示厂家ID，默认值是0
  * idProduct：表示产品ID，默认值是0

* 设置设备描述符，比如设置厂家ID、产品ID，这是可选的

  ```shell
  echo "0x1234" > idVendor
  echo "0x5678" > idProduct
  ```

* 创建配置：格式为"configs/<name>.<number>"，name可以取任意字符，number是配置编号

  ```shell
  mkdir configs/c.1
  ```

* 创建功能(function、接口)：格式为"functions/<name>.<instance name>"，name对应function的名字，比如acm对应ACM功能，对应的驱动为usb_f_acm.ko；instance name可以取任意字符

  ```shell
  mkdir functions/acm.test1
  ```

* 把配置和功能联系起来：ln -s functions/<name>.<instance name> configs/<name>.<number>

  ```shell
  ln -s functions/acm.test1  configs/c.1/
  ```

* 使能Gadget(确定使用哪个USB Device Controller)：echo <udc name> > UDC
  可用的UDC，可以在/sys/class/udc/*目录下查看

  ```shell
  echo ci_hdrc.0 > UDC
  ```

### 1.2 清除

* 禁止Gadget

  ```shell
  echo "" > UDC
  ```

* 移除配置里的功能(Remove functions from configurations)：
  命令：rm configs/<config name>.<number>/<function>

  ```shell
  rm  configs/c.1/acm.test1
  ```

* 移除配置：rmdir configs/<config name>.<number>

  ```shell
  rmdir configs/c.1
  ```

* 移除功能：rmdir functions/<name>.<instance name>

  ```shell
  rmdir functions/acm.test1
  ```

* 移除Gadget

  ```shell
  rmdir test_serial
  ```



### 1.3 STM32MP157上的实验

因为STM32MP157系统里已经使用的adb设备，要想模拟串口设备，需要先清除adb，命令如下：

```shell
cd /sys/kernel/config/usb_gadget/g1
echo "" > UDC
rm configs/b.1/ffs.adb
rmdir configs/b.1/strings/0x409
rmdir configs/b.1
rmdir functions/ffs.adb
rm strings/0x409
cd ..
rmdir g1
```

清除后，就按照《1.1 使用》来操作，需要注意的是最后一步：

```shell
ls  /sys/class/udc/
49000000.usb-otg

echo 49000000.usb-otg > UDC
```







## 2. 内部机制

### 2.1 configfs和sysfs

configfs和sysfs都是基于内存的虚拟文件系统，但是它们并不相同。

对于sysfs，当内核创建某个对象时，比如注册一个platform_drvier时，它就会被注册进sysfs里。它的属性就会在sysfs中出现：用户程序可以通过readdir、read函数读取这些属性，又是也可以通过write函数修改某些属性。重点在于：sysfs中的内容是在内核里创建、销毁，内核控制着sysfs的生命周期。可以认为sysfs就是这些内核对象的观察窗口。

对于configfs，当然也需要内核驱动程序的支撑。但是操作configfs的启动时用户程序：用户执行mkdir时会在内核里创建一个config_item对象，用户执行rmdir时会销毁这个内核对象。当执行mkdir创建目录时，这个config_item的属性就会出现在这个目录下。用户程序可以执行read、write操作读写这些属性。与sysfs的不同在于：configfs中目录、文件的生命周期由用户程序决定。

### 2.2 重要结构体

挂载configfs文件系统后，在里面创建/删除目录、读写文件、建立链接文件，都会导致内核中相关函数被调用。

站在用户的角度来说，一个文件系统里面有目录、文件两种对象。在configfs的内核实现中，对应4个概念。从底往上看：

* configfs_attribute、configfs_bin_attribute：对应文件

  * configfs_attribute对应的文件里含有的是可视化的字符串信息，它在内核里有一个结构体：

    ```c
    	struct configfs_attribute {
    		char                    *ca_name;
    		struct module           *ca_owner;
    		umode_t                  ca_mode;
    		ssize_t (*show)(struct config_item *, char *);
    		ssize_t (*store)(struct config_item *, const char *, size_t);
    	};
    ```

  * configfs_bin_attribute对应的文件里含有的是二进制信息，它在内核里有一个结构体：

    ```c
    struct configfs_bin_attribute {
    	struct configfs_attribute cb_attr;	/* std. attribute */
    	void *cb_private;			/* for user       */
    	size_t cb_max_size;			/* max core size  */
    	ssize_t (*read)(struct config_item *, void *, size_t);
    	ssize_t (*write)(struct config_item *, const void *, size_t);
    };
    ```

  * 读写文件时，会导致上述结构体里的show/store或者read/write函数被调用

  * 文件是位于某个目录的: config_item

* config_item：configfs中的每个对象都是config_item，后面的config_group、subsystem本质上都属于特殊的config_item

  * config_group、subsystem，config_item都对应一个目录

  * 跟config_group、subsystem对比时，config_item这个目录下不再有目录

  * 在config_item目录下有属性文件，还可以创建链接文件

  * 链接文件的操作结构体是：config_item_type里的configs_item_operations

    ![image-20230328101747270](pic/116_config_item.png)

    

* config_group：它是特殊的config_item，它有对应一个目录

  * 普通的config_item：下面不再有子目录
  * config_group：下面还可以创建config_item或者config_group，即：下面可以再创建子目录
  * 在当前目录下操作子目录时，对应的操作结构体是：config_item_type里的configs_group_operations
    ![image-20230328103059600](pic/117_config_group.png)

* subsystem：它是configfs文件系中的最顶层

  * 比如：`/sys/kernel/config/usb_gadget`、`/sys/kernel/config/iio`
  * 在`drivers\usb\gadget\configfs.c`中调用`configfs_register_subsystem(&gadget_subsys)`就会创建subsystem，它对应configfs文件系统中的顶层目录`usb_gadget`
  * subsystem也属于config_group
    ![image-20230328103329215](pic/118_subsystem.png)

### 2.3 configfs使用流程

跟legacy方法类比，要做的事情是一样的：

* 创建usb_composite_dev
* 设置设备描述符
* 设置配置描述符
* 添加接口（功能）

在视频里演示。