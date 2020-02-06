---
layout:     post
title:      "acpi summary"
subtitle:   "  "
date:       2020-02-06
author:     "Maggie"
header-img: "img/post-bg-digital-native.jpg"
catalog: true
tags:
    - linux
    - acpi
---



### ACPI 

ACPI 简介：

![](/img/in-post/post-acpi-summary1.bmp)



内存热插拔流程分析：

![](/img/in-post/post-acpi-memory-hotplug.svg)



#### ACPI

##### ACPI-General Event Device

patch中的介绍：

```c
Generic Event Device described in ACPI 6.1 allows platforms to handle
platform interrupts in ACPI ASL statements. It borrows constructs like
_EVT from GPIO events. All interrupts are listed in _CRS and the handler
is written in _EVT method. 

Device (GED0)
{

	Name (_HID, "ACPI0013")
	Name (_UID, 0)
	Name(_CRS, ResourceTemplate ()
	{
		Interrupt(ResourceConsumer, Edge, ActiveHigh, Shared, , , )
		 {123}
	})

	Method (_EVT, 1) {
		if (Lequal(123, Arg0))
		{
		}
	}
}
```

GED设备在ACPI的DSDT表中描述，所有的中断定义在`_CRS`中，中断触发的处理函数定义在`_EVT`中



看一下内核中的GED-dev的驱动初始化函数

```c
static int ged_probe(struct platform_device *pdev)
{
    acpi_status acpi_ret;

    acpi_ret = acpi_walk_resources(ACPI_HANDLE(&pdev->dev), "_CRS",
                       acpi_ged_request_interrupt, &pdev->dev);
    if (ACPI_FAILURE(acpi_ret)) {
        dev_err(&pdev->dev, "unable to parse the _CRS record\n");
        return -EINVAL;
    }

    return 0;
}
```

驱动函数会遍历DSDT表中GED设备的_CRS资源（中断信息），得到irq注册中断并注册中断处理函数（EVT中定义的函数）。中断处理函数会发送SCI中断到GuestOS。



ACPI表的解析过程

* 结构

  ![](/img/in-post/post-acpi-summary2.bmp)

* 解析过程

  https://blog.csdn.net/woai110120130/article/details/93318611