# Gadget驱动程序框架 #

参考资料：

* Linux下USB gadget设备详解：https://www.docin.com/p-1852320293.html
* Linux usb gadget框架概述：https://blog.csdn.net/daocaokafei/article/details/114114824
* USB设备驱动程序-USB Gadget Driver(二)：https://blog.csdn.net/chenzhen1080/article/details/53742924
* usb gadge驱动设计之我是zero：https://www.bbsmax.com/A/Ae5RvbwM5Q/
* Linux-USB驱动笔记（四）--USB整体框架: https://qingmu.blog.csdn.net/article/details/119979199
* USB gadget设备驱动解析:  https://www.likecs.com/show-843861.html
* 调试软件：USB view 、bushound 、及一些硬件USB信号分析仪
* 可以用[wireshark](https://so.csdn.net/so/search?q=wireshark&spm=1001.2101.3001.7020)+usbmon捕捉usb协议数据包。



## 1. 笔记

Tutorial about USB HID Report Descriptors

https://eleccelerator.com/tutorial-about-usb-hid-report-descriptors/

USB HID Report Descriptor 报告描述符详解 - Climber丶 - 博客园.html

https://www.cnblogs.com/AlwaysOnLines/p/3859557.html



![image-20221110164557862](../../../../linux_basic_develop/05_drivers/imx6ull_pro/doc/pic/81_udc_bind_to_driver.png)



```shell
struct usb_gadget - represents a usb slave device
struct usb_composite_device - represents one composite usb gadget
struct usb_udc - describes one usb device controller
```



```c
/**
 * struct usb_udc - describes one usb device controller
 * @driver - the gadget driver pointer. For use by the class code
 * @dev - the child device to the actual controller
 * @gadget - the gadget. For use by the class code
 * @list - for use by the udc class driver
 * @vbus - for udcs who care about vbus status, this value is real vbus status;
 * for udcs who do not care about vbus status, this value is always true
 *
 * This represents the internal data structure which is used by the UDC-class
 * to hold information about udc driver and gadget together.
 */
struct usb_udc {
	struct usb_gadget_driver	*driver;
	struct usb_gadget		*gadget;
	struct device			dev;
	struct list_head		list;
	bool				vbus;
};


// usb_udc用来表示USB设备控制器，它里面有：
// 1. usb_gadget : 用来表示硬件信息？represents a usb slave device，比如USB串口？
// 2. usb_gadget_driver: 用来表示软件信息？driver for usb 'slave' devices
```



usb_gadget和usb_composite_dev必定一起出现：usb_gadget描述的是硬件底层信息(?)，usb_composite_dev描述的是硬件的上层信息(？)

![image-20221115145504202](pic/78_gadget_composite_dev.png)



### 1.1 结构体

```c
struct usb_udc - describes one usb device controller
struct usb_gadget - represents a usb slave device
struct usb_gadget_driver - driver for usb 'slave' devices
struct usb_composite_device - represents one composite usb gadget
struct usb_function - describes one function of a configuration
struct usb_composite_driver - groups configurations into a gadget
```





### 1.2 UDC驱动分析

```shell
/home/book/100ask_imx6ull-sdk/Linux-4.9.88/drivers/usb/chipidea/ci_hdrc_imx.c
	data->ci_pdev = ci_hdrc_add_device(dev,
				pdev->resource, pdev->num_resources,
				&pdata);

/home/book/100ask_imx6ull-sdk/Linux-4.9.88/drivers/usb/chipidea/core.c
static struct platform_driver ci_hdrc_driver = {
	.probe	= ci_hdrc_probe,
	.remove	= ci_hdrc_remove,
	.driver	= {
		.name	= "ci_hdrc",
		.pm	= &ci_pm_ops,
	},
};
```

![image-20221117145147166](pic/79_imx_otg.png)



中断：

![image-20221118102514064](pic/80_interrupt_imx6ull_otg.png)





#### 1.2.1 注册udc

```c
// drivers/usb/chipidea/core.c
ci_hdrc_probe
	ret = ci_hdrc_gadget_init(ci);
				struct ci_role_driver *rdrv;
                rdrv->start	= udc_id_switch_for_device;
                rdrv->stop	= udc_id_switch_for_host;
                rdrv->irq	= udc_irq;
                rdrv->suspend	= udc_suspend;
                rdrv->resume	= udc_resume;
                rdrv->name	= "gadget";

                ret = udc_start(ci);	
                        ci->gadget.ops          = &usb_gadget_ops;
                        ci->gadget.speed        = USB_SPEED_UNKNOWN;
                        ci->gadget.max_speed    = USB_SPEED_HIGH;
                        ci->gadget.name         = ci->platdata->name;
                        ci->gadget.otg_caps	= otg_caps;

                        retval = init_eps(ci);
                                    hwep->ep.name      = hwep->name;
                                    hwep->ep.ops       = &usb_ep_ops;
                        if (retval)
                            goto free_pools;

                        ci->gadget.ep0 = &ci->ep0in->ep;

						retval = usb_add_gadget_udc(dev, &ci->gadget);
									usb_add_gadget_udc_release(parent, gadget, NULL);
										list_add_tail(&udc->list, &udc_list);
                if (!ret)
                    ci->roles[CI_ROLE_GADGET] = rdrv;

    platform_set_drvdata(pdev, ci);
    ret = devm_request_irq(dev, ci->irq, ci_irq, IRQF_SHARED,
                           ci->platdata->name, ci);
    if (ret)
        goto stop;
```



#### 1.2.2 UDC中断处理函数

```c
// /home/book/100ask_imx6ull-sdk/Linux-4.9.88/drivers/usb/chipidea/core.c
ci_irq
	/* Handle device/host interrupt */
	if (ci->role != CI_ROLE_END)
		ret = ci_role(ci)->irq(ci);    
					udc_irq
                        if (USBi_UI  & intr)
                            isr_tr_complete_handler(ci);      

                                /* Only handle setup packet below */
                                if (i == 0 &&
                                    hw_test_and_clear(ci, OP_ENDPTSETUPSTAT, BIT(0)))
                                    isr_setup_packet_handler(ci);
```



#### 1.2.3 控制传输的处理

isr_setup_packet_handler对于ep0的处理，

有很多x消息并不需要传递到最上层的composite驱动，

这个函数的前面代码就自己处理了这些setup包，

后面无法处理的request才交给上层

```c
static void isr_setup_packet_handler(struct ci_hdrc *ci)
__releases(ci->lock)
__acquires(ci->lock)
{
	struct ci_hw_ep *hwep = &ci->ci_hw_ep[0];
	struct usb_ctrlrequest req;
	int type, num, dir, err = -EINVAL;
	u8 tmode = 0;

	/*
	 * Flush data and handshake transactions of previous
	 * setup packet.
	 */
	_ep_nuke(ci->ep0out);
	_ep_nuke(ci->ep0in);

	/* read_setup_packet */
	do {
		hw_test_and_set_setup_guard(ci);
		memcpy(&req, &hwep->qh.ptr->setup, sizeof(req));
	} while (!hw_test_and_clear_setup_guard(ci));

	type = req.bRequestType;

	ci->ep0_dir = (type & USB_DIR_IN) ? TX : RX;

	switch (req.bRequest) {
	case USB_REQ_CLEAR_FEATURE:
		if (type == (USB_DIR_OUT|USB_RECIP_ENDPOINT) &&
				le16_to_cpu(req.wValue) ==
				USB_ENDPOINT_HALT) {
			if (req.wLength != 0)
				break;
			num  = le16_to_cpu(req.wIndex);
			dir = (num & USB_ENDPOINT_DIR_MASK) ? TX : RX;
			num &= USB_ENDPOINT_NUMBER_MASK;
			if (dir == TX)
				num += ci->hw_ep_max / 2;
			if (!ci->ci_hw_ep[num].wedge) {
				spin_unlock(&ci->lock);
				err = usb_ep_clear_halt(
					&ci->ci_hw_ep[num].ep);
				spin_lock(&ci->lock);
				if (err)
					break;
			}
			err = isr_setup_status_phase(ci);
		} else if (type == (USB_DIR_OUT|USB_RECIP_DEVICE) &&
				le16_to_cpu(req.wValue) ==
				USB_DEVICE_REMOTE_WAKEUP) {
			if (req.wLength != 0)
				break;
			ci->remote_wakeup = 0;
			err = isr_setup_status_phase(ci);
		} else {
			goto delegate;
		}
		break;
	case USB_REQ_GET_STATUS:
		if ((type != (USB_DIR_IN|USB_RECIP_DEVICE) ||
			le16_to_cpu(req.wIndex) == OTG_STS_SELECTOR) &&
		    type != (USB_DIR_IN|USB_RECIP_ENDPOINT) &&
		    type != (USB_DIR_IN|USB_RECIP_INTERFACE))
			goto delegate;
		if ((le16_to_cpu(req.wLength) != 2 &&
			le16_to_cpu(req.wLength) != 1) ||
				le16_to_cpu(req.wValue) != 0)
			break;
		err = isr_get_status_response(ci, &req);
		break;
	case USB_REQ_SET_ADDRESS:
		if (type != (USB_DIR_OUT|USB_RECIP_DEVICE))
			goto delegate;
		if (le16_to_cpu(req.wLength) != 0 ||
		    le16_to_cpu(req.wIndex)  != 0)
			break;
		ci->address = (u8)le16_to_cpu(req.wValue);
		ci->setaddr = true;
		err = isr_setup_status_phase(ci);
		break;
	case USB_REQ_SET_FEATURE:
		if (type == (USB_DIR_OUT|USB_RECIP_ENDPOINT) &&
				le16_to_cpu(req.wValue) ==
				USB_ENDPOINT_HALT) {
			if (req.wLength != 0)
				break;
			num  = le16_to_cpu(req.wIndex);
			dir = (num & USB_ENDPOINT_DIR_MASK) ? TX : RX;
			num &= USB_ENDPOINT_NUMBER_MASK;
			if (dir == TX)
				num += ci->hw_ep_max / 2;

			spin_unlock(&ci->lock);
			err = _ep_set_halt(&ci->ci_hw_ep[num].ep, 1, false);
			spin_lock(&ci->lock);
			if (!err)
				isr_setup_status_phase(ci);
		} else if (type == (USB_DIR_OUT|USB_RECIP_DEVICE)) {
			if (req.wLength != 0)
				break;
			switch (le16_to_cpu(req.wValue)) {
			case USB_DEVICE_REMOTE_WAKEUP:
				ci->remote_wakeup = 1;
				err = isr_setup_status_phase(ci);
				break;
			case USB_DEVICE_TEST_MODE:
				tmode = le16_to_cpu(req.wIndex) >> 8;
				switch (tmode) {
				case TEST_J:
				case TEST_K:
				case TEST_SE0_NAK:
				case TEST_PACKET:
				case TEST_FORCE_EN:
					ci->test_mode = tmode;
					err = isr_setup_status_phase(
							ci);
					break;
				case TEST_OTG_SRP_REQD:
					err = otg_srp_reqd(ci);
					break;
				case TEST_OTG_HNP_REQD:
					err = otg_hnp_reqd(ci);
					break;
				default:
					break;
				}
				break;
			case USB_DEVICE_B_HNP_ENABLE:
				if (ci_otg_is_fsm_mode(ci)) {
					ci->gadget.b_hnp_enable = 1;
					err = isr_setup_status_phase(
							ci);
				}
				break;
			case USB_DEVICE_A_ALT_HNP_SUPPORT:
				if (ci_otg_is_fsm_mode(ci))
					err = otg_a_alt_hnp_support(ci);
				break;
			case USB_DEVICE_A_HNP_SUPPORT:
				if (ci_otg_is_fsm_mode(ci)) {
					ci->gadget.a_hnp_support = 1;
					err = isr_setup_status_phase(
							ci);
				}
				break;
			default:
				goto delegate;
			}
		} else {
			goto delegate;
		}
		break;
	default:
delegate:
		if (req.wLength == 0)   /* no data phase */
			ci->ep0_dir = TX;

		spin_unlock(&ci->lock);
		err = ci->driver->setup(&ci->gadget, &req);
            	// composite_driver_template.setup
            	composite_setup
		spin_lock(&ci->lock);
		break;
	}

	if (err < 0) {
		spin_unlock(&ci->lock);
		if (_ep_set_halt(&hwep->ep, 1, false))
			dev_err(ci->dev, "error: _ep_set_halt\n");
		spin_lock(&ci->lock);
	}
}

```





#### 1.2.4 EP什么时候使能

```c
static int udc_bind_to_driver(struct usb_udc *udc, struct usb_gadget_driver *driver)
{
	int ret;

	dev_dbg(&udc->dev, "registering UDC driver [%s]\n",
			driver->function);

	udc->driver = driver;
	udc->dev.driver = &driver->driver;
	udc->gadget->dev.driver = &driver->driver;

	ret = driver->bind(udc->gadget, driver);
	if (ret)
		goto err1;
	ret = usb_gadget_udc_start(udc);
    			return udc->gadget->ops->udc_start(udc->gadget, udc->driver);
    						ci_udc_start
                                
```

![image-20221118144034285](pic/81_ci_udc_start.png)





## 