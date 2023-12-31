# Gadget应用实例之serial #

参考资料：

* https://blog.csdn.net/embededswordman/article/details/6689593
* Linux文档：Documentation\usb\gadget_serial.txt
* UART教程：GIT仓库`doc_and_source_for_drivers\IMX6ULL\doc_pic\09_UART`
  * 回顾TTY的概念
  * 回顾console的概念



## 1. 硬件体验

使用USB线连接板子的OTG口和PC的USB口。

然后在板子加载驱动程序后，可以看到新的设备节点/dev/ttyGS0：

```shell
# modprobe g_serial
g_serial gadget: Gadget Serial v2.4
g_serial gadget: g_serial ready
g_serial gadget: high-speed config #2: CDC ACM config

# ls /dev/ttyGS0 -l
crw-rw----    1 root     dialout   246,   0 Jan  1 00:30 /dev/ttyGS0
```



在PC上，如果是Windows系统，可以在设备管理器里看到新的USB串口：

![image-20230306113121976](pic/113_gadget_serial_on_windows.png)



在PC上，如果是VMware上的Linux系统，按下图操作，先把USB串口连接到VMware：

![image-20230306113733792](pic/114_connect_usb_serial_to_vmware.png)



然后在PC Linux中可以看到新的设备节点：

```shell
book@100ask:~$ dmesg
[  286.903239] usb 1-1: new high-speed USB device number 2 using ehci-pci
[  287.254549] usb 1-1: New USB device found, idVendor=0525, idProduct=a4a7, bcdDevice= 4.09
[  287.254550] usb 1-1: New USB device strings: Mfr=1, Product=2, SerialNumber=0
[  287.254551] usb 1-1: Product: Gadget Serial v2.4
[  287.254552] usb 1-1: Manufacturer: Linux 4.9.88 with 2184000.usb
[  287.342786] cdc_acm 1-1:2.0: ttyACM0: USB ACM device
[  287.343202] usbcore: registered new interface driver cdc_acm
[  287.343202] cdc_acm: USB Abstract Control Model driver for USB modems and ISDN adapters
book@100ask:~$ ls /dev/ttyACM0 -l
crw-rw---- 1 root dialout 166, 0 Mar  5 22:38 /dev/ttyACM0
```



## 2. Serial分析

### 2.1 软件框架

Gadget串口的框架如下：

![image-20230306112530219](pic/108_usb_serial_gadget_frame.png)

u_serial提供了有2种方法来使用Gadget串口：

* u_serial.c里注册tty_driver结构体gs_tty_driver，在板子上编写APP访问设备/dev/ttyGS0即可与Host交互(Host要打开USB串口)
  ![image-20230306105115767](pic/110_gs_tty_driver.png)

* u_serial.c里注册console结构体gserial_cons。启动Linux内核时传入commandline参数"console=ttyGS0"后，内核的printk的信息通过Gadget串口打印出来(Host要打开USB串口)：

  ![image-20230306105043886](pic/109_gserial_cons.png)



注册TTY和console的过程：

```shell
gs_bind // drivers\usb\gadget\legacy\serial.c
    status  = serial_register_ports(cdev, &serial_config_driver,
		    "acm");
		    
		    	fi_serial[i] = usb_get_function_instance(f_name);
	
	
acm_alloc_instance // drivers\usb\gadget\function\f_acm.c
	ret = gserial_alloc_line(&opts->port_num); // drivers\usb\gadget\function\u_serial.c
	
			// 注册TTY
            tty_dev = tty_port_register_device(&ports[port_num].port->port,
                    gs_tty_driver, port_num, NULL);

			// 注册console
			gserial_console_init();
            	register_console(&gserial_cons);
	
```





### 2.2 数据传输

#### 2.2.1 APP访问

注意，在USB中数据传输总是由Host发起，所以：

* 板子要事先准备好空间(设置好out方向的usb_request并放入队列)，以便接收Host发来的数据；
* 板子有数据想发送给Host时需要设置in方向的usb_request，以便Host读取。

板子上的APP访问/dev/ttyGS0时，就会导致gs_tty_ops结构体的对应函数被调用：

![image-20230306110254302](pic/111_gs_tty_ops.png)

APP调用open函数时，会导致如下调用：

```shell
gs_open
	gs_start_io(port);
		// 取出out端点(对应Host来说是out, 对于板子来说就是输入)
		struct usb_ep		*ep = port->port_usb->out;
		
		// 给out端点分配usb_request
        status = gs_alloc_requests(ep, head, gs_read_complete,
            &port->read_allocated);

		// 给in端点分配usb_request, 但是在open时并没有把in方向的usb_request放入队列
        status = gs_alloc_requests(port->port_usb->in, &port->write_pool,
                gs_write_complete, &port->write_allocated);

        // 把usb_request放入队列, 如果Host发来数据, 这个usb_request的complete函数被调用
		started = gs_start_rx(port);
					status = usb_ep_queue(out, req, GFP_ATOMIC);
```



APP调用write函数时，会导致如下调用：

```shell
gs_write
	gs_start_tx(port);
		// 把usb_request放入队列, Host读取数据时就可以从中得到数据
		status = usb_ep_queue(in, req, GFP_ATOMIC);
```



#### 2.2.2 printk

启动Linux内核时传入commandline参数"console=ttyGS0"后，内核的printk的信息通过Gadget串口打印出来(Host要打开USB串口)。

内核的printk函数会导致gserial_cons结构体中的write指针即`gs_console_write`函数被调用：

![image-20230306111944494](pic/112_gserial_cons_write.png)

gs_console_write函数的调用关系如下：

```shell
gs_console_write
	// 把要打印的数据放入环形buffer
	gs_buf_put(&info->con_buf, buf, count);
	
	// 唤醒内核线程
	wake_up_process(info->console_thread);
	
// 内核线程
gs_console_thread
	// 被唤醒后
	
	// 取出输入端点和它的usb_request
	req = info->console_req;
	ep = port->port_usb->in;
	
	// 从环形buffer得到数据、设置usb_request
	xfer = gs_buf_get(&info->con_buf, req->buf, size);
	req->length = xfer;
	
	// 把usb_request放入队列，以便Host读取
	ret = usb_ep_queue(ep, req, GFP_ATOMIC);
```



## 3. 编程

PC: open/read/write  /dev/ttyACM0

板子: open/read/write  /dev/ttyGS0

参考资料：https://stackoverflow.com/questions/7469139/what-is-the-equivalent-to-getch-getche-in-linux

源码：

![image-20230306150940582](pic/115_gadget_serial_src.png)

## 4. 上机实验

编译2个版本：PC、ARM

```shell
gcc -o serial_send_recv_pc serial_send_recv.c -lpthread
arm-buildroot-linux-gnueabihf-gcc -o serial_send_recv_arm serial_send_recv.c -lpthread
```



使用USB线连接板子的OTG口、PC的USB口，PC上监测到USB串口后把它连接到VMWare，确定：

* 开发板上有设备节点：/dev/ttyGS0
* Ubuntu上有设备节点：/dev/ttyACM0



测试：

* 在Ubuntu上执行：`sudo  ./serial_send_recv_pc  /dev/ttyACM0`
* 在板子上执行：`sudo  ./serial_send_recv_arm  /dev/ttyGS0`
* 双方即可互发数据



