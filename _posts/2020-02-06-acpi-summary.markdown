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

术语：

```c
ASL        : ACPI Source Language

AML        : ACPI Machine Language

DSL        : Digital Simulation Language

E820       : a system memory map protocol, provided in ACPI spec, ch14 for 3.0b

EFI        : Enhanced Firmware Interface

HPET       : High Precision Event Timer

GPE        : General-Purpose Event

GSI        : Global System Interrupts

OSL        : OS Service Layer

OSPM       : OS Power Management, 指Linux等OS中实现对ACPI支持的代码
						// OS中已驱动程序的方式控制ACPI模块，并可以处理热插拔、SCI中断和设备事件等
PRT        : PCI IRQ Routing Table

PXE        : Preboot Execution Environment

SAPIC      : Streamlined APIC, IA64上使用的APIC。其local SAPIC和I/O SAPIC分别对 应着IA32 和x86-64上的Local APIC和I/O APIC

SBF        : Simple Boot Flag

SCI        : System Control Interrupt  (OS-visible interrupts, triggered by ACPI events)

SMBIOS/DMI : System Management BIOS/Desktop Management Interface. PC的BIOS规范。

TOM        : Top Of Memory

UUID       : Universal Uniform IDentifiers

xface      : Linux内核ACPI源文件的命名法，表示Interface。例如tbxface.c实现table的接口。
```



ACPI 简介：

![](/img/in-post/post-acpi-summary1.bmp)



内存热插拔流程分析：

![](/img/in-post/post-acpi-memory-hotplug.svg)





#### ACPI core system

![](/img/in-post/acpi-core-system-module.bmp)

* AML interpreter

  在core system中，AML interpreter是关键，负责分析和运行本机BIOS中得到的AML文件流



* Table management

  载入（载入过程中会分析和校验）和管理来自BIOS的acpi-tables

* namespace management

  负责载入和管理namespace

* Resource Management

  资源管理提供建立在名字空间资源的配置和获取，其中包括了 PCI的设备的地址区间，中断等重要参数。它所提供的服务包括：获取和设定当前的资源，获取设备上可能存在的地址区间以及 PCI 设备的中断路由表（IRQ Routing Tables），获取当前设备的电源支持能力（例如是否支持 S1-S5 状态）

* ACPI H/W Management

  该模块用于控制对桥芯片上 ACPI 寄存器和时钟以及其他 ACPI 关联硬件的访问，例如 ACPI GPE 状态寄存器和使能寄存器，系统状态获得。 

* Event handling：事件管理模块是用于管理系统控制中断（SCI）的发生和 GPE 事件的响应，SCI 包括 ACPI 时钟中断，以及 GPE 事件管理。这个单元负责"分发"地址空间和操作空间（OperationRegion）的事件到当前的操作系统层，并负责调用相关的句柄来进行处理。

#### OS Service layer

* 提供OS和ACPI交互的接口
* ACPI核心服务层可以将ACPI核心层的服务转换成OS内部的调用。OS向ACPI发出的请求也必须通过OSL







### 初始化

#### ACPI  General Purpose Event

> Four types of general purpose events are supported:
>
> - GPEs that are defined by a GPE block described within the FADT.
>
> - GPEs that are defined by a GPE Block Device.
>
> - GPIO-signaled events that are defined by _AEI object of the GPIO controller device
>
> - Interrupt-signaled events that are defined by _CRS object of the Generic Event Device (GED)
>
>   
>
>   The four types of events are differentiated by the type of the *EventInfo* object in the returned package. For FADT-based GPEs, *EventInfo* is an **Integer** containing a bit index. For Block Device- based GPEs, *EventInfo* is a **Package** containing a **Reference** to the parent block device and an **Integer** containing a bit index. For GPIO-signaled events, *EventInfo* is a **Package** containing a Reference to the GPIO controller device and an **Integer** containing the index of the event in the _AEI object (starting from zero). For Interrupt-signaled events, *EventInfo* is a **Package** containing a **Reference** to the GED and an **Integer** containing the index of the event in the _CRS object (starting from zero).
>
> ——acpi spec 6

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

  ![](/img/in-post/acpi_table.bmp)
  ![](/img/in-post/fadt_table.bmp)

* 解析过程

  https://blog.csdn.net/woai110120130/article/details/93318611







#### ACPI table解析过程



与ACPI相关的全局变量：

```c
/* Master list of all ACPI tables that were found in the RSDT/XSDT */

ACPI_GLOBAL(struct acpi_table_list, acpi_gbl_root_table_list);

/* DSDT information. Used to check for DSDT corruption */

ACPI_GLOBAL(struct acpi_table_header *, acpi_gbl_DSDT);
ACPI_GLOBAL(struct acpi_table_header, acpi_gbl_original_dsdt_header);
ACPI_INIT_GLOBAL(u32, acpi_gbl_dsdt_index, ACPI_INVALID_TABLE_INDEX);
ACPI_INIT_GLOBAL(u32, acpi_gbl_facs_index, ACPI_INVALID_TABLE_INDEX);
ACPI_INIT_GLOBAL(u32, acpi_gbl_xfacs_index, ACPI_INVALID_TABLE_INDEX);
ACPI_INIT_GLOBAL(u32, acpi_gbl_fadt_index, ACPI_INVALID_TABLE_INDEX);
```

ACPI-table的表头：

```c
/*******************************************************************************
 *
 * Master ACPI Table Header. This common header is used by all ACPI tables
 * except the RSDP and FACS.
 *
 ******************************************************************************/

struct acpi_table_header {
    char signature[ACPI_NAME_SIZE]; /* ASCII table signature */
    u32 length;     /* Length of table in bytes, including this header */
    u8 revision;        /* ACPI Specification minor version number */
    u8 checksum;        /* To make sum of entire table == 0 */
    char oem_id[ACPI_OEM_ID_SIZE];  /* ASCII OEM identification */  // 厂商id
    char oem_table_id[ACPI_OEM_TABLE_ID_SIZE];  /* ASCII OEM table identification */ // table id
    u32 oem_revision;   /* OEM revision number */
    char asl_compiler_id[ACPI_NAME_SIZE];   /* ASCII ASL compiler vendor ID */
    u32 asl_compiler_revision;  /* ASL compiler version */
};
```

 

下面看一下acpi_tb_parse_root_table函数：

其中结构体`acpi_table_rsdp`中保存了RSDT-table的物理地址（32位）和XSDT-table的物理地址（64位）

```c
/*******************************************************************************
 *
 * FUNCTION:    acpi_tb_parse_root_table
 *
 * PARAMETERS:  rsdp_address        - Pointer to the RSDP
 *
 * RETURN:      Status
 *
 * DESCRIPTION: This function is called to parse the Root System Description
 *              Table (RSDT or XSDT)
 *
 * NOTE:        Tables are mapped (not copied) for efficiency. The FACS must
 *              be mapped and cannot be copied because it contains the actual
 *              memory location of the ACPI Global Lock.
 *
 ******************************************************************************/

acpi_status ACPI_INIT_FUNCTION
acpi_tb_parse_root_table(acpi_physical_address rsdp_address)
{
	struct acpi_table_rsdp *rsdp;
	u32 table_entry_size;
	u32 i;
	u32 table_count;
	struct acpi_table_header *table;
	acpi_physical_address address;
	u32 length;
	u8 *table_entry;
	acpi_status status;
	u32 table_index;

	ACPI_FUNCTION_TRACE(tb_parse_root_table);

	/* Map the entire RSDP and extract the address of the RSDT or XSDT */
	rsdp = acpi_os_map_memory(rsdp_address, sizeof(struct acpi_table_rsdp));
	if (!rsdp) {
		return_ACPI_STATUS(AE_NO_MEMORY);
	}

	acpi_tb_print_table_header(rsdp_address,
				   ACPI_CAST_PTR(struct acpi_table_header,
						 rsdp));

	/* Use XSDT if present and not overridden. Otherwise, use RSDT */
// 根据ACPI版本的不同，选择使用RSDT或者XSDT，版本1 用RSDT， Reversion > 1 用XSDT，
// 两者 的区别在于 XSDT是64位的，RSDT是32位的
	if ((rsdp->revision > 1) &&
	    rsdp->xsdt_physical_address && !acpi_gbl_do_not_use_xsdt) {
		/*
		 * RSDP contains an XSDT (64-bit physical addresses). We must use
		 * the XSDT if the revision is > 1 and the XSDT pointer is present,
		 * as per the ACPI specification.
		 */
		address = (acpi_physical_address)rsdp->xsdt_physical_address;
		table_entry_size = ACPI_XSDT_ENTRY_SIZE;
	} else {
		/* Root table is an RSDT (32-bit physical addresses) */

		address = (acpi_physical_address)rsdp->rsdt_physical_address;
		table_entry_size = ACPI_RSDT_ENTRY_SIZE;
	}

	/*
	 * It is not possible to map more than one entry in some environments,
	 * so unmap the RSDP here before mapping other tables
	 */
	acpi_os_unmap_memory(rsdp, sizeof(struct acpi_table_rsdp));

	/* Map the RSDT/XSDT table header to get the full table length */

	table = acpi_os_map_memory(address, sizeof(struct acpi_table_header));
	if (!table) {
		return_ACPI_STATUS(AE_NO_MEMORY);
	}

	acpi_tb_print_table_header(address, table);

	/*
	 * Validate length of the table, and map entire table.
	 * Minimum length table must contain at least one entry.
	 */
	length = table->length;
	acpi_os_unmap_memory(table, sizeof(struct acpi_table_header));

	if (length < (sizeof(struct acpi_table_header) + table_entry_size)) {
		ACPI_BIOS_ERROR((AE_INFO,
				 "Invalid table length 0x%X in RSDT/XSDT",
				 length));
		return_ACPI_STATUS(AE_INVALID_TABLE_LENGTH);
	}

	table = acpi_os_map_memory(address, length);
	if (!table) {
		return_ACPI_STATUS(AE_NO_MEMORY);
	}

	/* Validate the root table checksum */
// 验证表校验和
	status = acpi_tb_verify_checksum(table, length);
	if (ACPI_FAILURE(status)) {
		acpi_os_unmap_memory(table, length);
		return_ACPI_STATUS(status);
	}

	/* Get the number of entries and pointer to first entry */

	table_count = (u32)((table->length - sizeof(struct acpi_table_header)) /
			    table_entry_size);
	table_entry = ACPI_ADD_PTR(u8, table, sizeof(struct acpi_table_header));

	/* Initialize the root table array from the RSDT/XSDT */

	for (i = 0; i < table_count; i++) {

		/* Get the table physical address (32-bit for RSDT, 64-bit for XSDT) */

		address =
		    acpi_tb_get_root_table_entry(table_entry, table_entry_size); // 获取一个entry

		/* Skip NULL entries in RSDT/XSDT */

		if (!address) {
			goto next_table;
		}

    // 安装标准的table
		status = acpi_tb_install_standard_table(address,
							ACPI_TABLE_ORIGIN_INTERNAL_PHYSICAL,
							FALSE, TRUE,
							&table_index);

    // 如果是FADT，调用acpi_tb_parse_fadt() 进行解析
		if (ACPI_SUCCESS(status) &&
		    ACPI_COMPARE_NAMESEG(&acpi_gbl_root_table_list.
					 tables[table_index].signature,
					 ACPI_SIG_FADT)) {
			acpi_gbl_fadt_index = table_index;
			acpi_tb_parse_fadt();
		}

next_table:

		table_entry += table_entry_size;
	}

	acpi_os_unmap_memory(table, length);
	return_ACPI_STATUS(AE_OK);
}
```



函数`acpi_tb_install_standard_table`完成的功能： 

* `acpi_tb_acquire_temp_table` 用于创建一个acpi_table_desc结构，acpi_table_desc定义如下

  ```c
  struct acpi_table_desc {
      acpi_physical_address address;  // 物理地址
      struct acpi_table_header *pointer;  // table头地址
      u32 length;     /* Length fixed at 32 bits (fixed in table header) */
      union acpi_name_union signature;
      acpi_owner_id owner_id;
      u8 flags;
  };
  ```

* `acpi_tb_verify_temp_table`验证table是否可用，主要验证包括签名，check sum，长度等信息

* reload表示重新加载，要先卸载原来的再加载当前的

* `acpi_tb_install_table_with_override` 安装： 会将表的相关信息记录到全局变量acpi_gbl_root_table_list中



**FADT table的解析`acpi_tb_parse_fadt`:**

解析FADT表和子表DSDT-table和FACS-table，同时设置acpi相关全局变量

```c
acpi_tb_parse_fadt
  // 设置acpi_gbl_DSDT全局变量
  // 设置表在全局变量acpi_gbl_root_table_list中的index
  // acpi_gbl_dsdt_index、acpi_gbl_facs_index、acpi_gbl_xfacs_index
	-> acpi_tb_install_standard_table
		-> acpi_tb_install_table_with_override
```



#### ACPI table解析时机

```c
// kernel 
start_kernel
	-> setup_arch
		-> acpi_boot_table_init
			-> acpi_table_init
				-> acpi_initialize_tables
  				// 获取rsdp地址
  				-> rsdp_address = acpi_os_get_root_pointer() 
					-> acpi_tb_parse_root_table
```



#### ACPI namepspace

---

ACPI-initialization  work flow:

start_kernel

​	-> setup_arch()  ... -> acpi_tb_parse_root_table // 解析ACPI-table

​	-> acpi_early_init()

​	// 在start_kernel() 的最后

​	-> acpi_init_subsystem() // 设置ACPI mode

​	-> arch_call_rest_init ...-> acpi_init

---

* 基本

​	namespace是一个树状层次机构，在受操作系统控制的内存里面，这段内存里面包含命名对象（named objects）等。这些对象（objects）可以是**数据对象，控制方法对象，总线/设备包对象**等。操作系统通过从驻留在 ACPI BIOS 中的 ACPI Tables 载入载出（loading and/or unloading）定义块（definition blocks），来动态改变命名空间（namespace）的内容。在ACPI Namespace 中的所有信息都来自 Differentiated System Description Table (DSDT)，DSDT 里面包含了 Differentiated Definition Block 还有一个或者多个其他的定义块（definition blocks）。



* 在Runtime阶段，ACPI通过load和unload ACPI-tables中的Definition Blocks来修改namespace中的内容

* OS 枚举主板上的设备，通过读取namespace中的包含__HID的devices

* 预定义的namespace

  右斜杠\是根目录，后面的点.标识当前父节点的子节点

  ```
  \_GPE        : General events in GPE register block
  
  \_PR        : ACPI 1.0 Processor Namespace.
  
  \_SB        : All Device/Bus Objects under this namespace
  
  \_SI        : System Indicator.
  
  \_TZ        : ACPI 1.0 Thermal Zonen namespace.
  ```



* 内核中定义的namespace全局变量

  ```c
  ACPI_GLOBAL(struct acpi_namespace_node, acpi_gbl_root_node_struct);
  ACPI_GLOBAL(struct acpi_namespace_node *, acpi_gbl_root_node);
  ACPI_GLOBAL(struct acpi_namespace_node *, acpi_gbl_fadt_gpe_device);
  ```

  

##### 解析时机

ACPI-namespace解析的入口为

```c 
acpi_init()
	-> acpi_bus_init
		-> acpi_load_tables
			-> acpi_tb_laod_namespace
```

下面看一下acpi_init的定义：

```c
static int __init acpi_init(void)
{
	int result;

	if (acpi_disabled) {
		printk(KERN_INFO PREFIX "Interpreter disabled.\n");
		return -ENODEV;
	}

	acpi_kobj = kobject_create_and_add("acpi", firmware_kobj);
	if (!acpi_kobj) {
		printk(KERN_WARNING "%s: kset create error\n", __func__);
		acpi_kobj = NULL;
	}

	result = acpi_bus_init();
	if (result) {
		disable_acpi();
		return result;
	}

	pci_mmcfg_late_init();
	acpi_iort_init();
	acpi_scan_init();
	acpi_ec_init();
	acpi_debugfs_init();
	acpi_sleep_proc_init();
	acpi_wakeup_device_init();
	acpi_debugger_init();
	acpi_setup_sb_notify_handler();
	return 0;
}
// 注意这里将该函数放在特定的段里
subsys_initcall(acpi_init);
```

许多的子系统都有自己的初始化函数，而这些初始化的函数又根据功能不同被分开在不同的子段里，子段的排列顺序则由链接决定。为了向后兼容，initcall()把调用，也就是一个个指向初始化函数的函数指针放进设备初始化子段里（[参考1](https://www.cnblogs.com/sky-heaven/p/5388137.html)，[参考2](https://blog.csdn.net/fenzhikeji/article/details/6860143)）。这些段的函数调用链为：

```c
// kernel 
start_kernel
	-> setup_arch
		-> acpi_boot_table_init
			-> acpi_table_init
				-> acpi_initialize_tables
					-> acpi_tb_parse_root_table
  -> arch_call_rest_init()
  	-> rest_init()
  		-> (new thread) kernel_thread(kernel_init, NULL, CLONE_FS);
				-> kernel_init_freeable
          -> do_basic_setup
          	-> do_initcalls
 
// 执行各个init_call的函数
static void __init do_initcalls(void)
{
	int level;

	for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
		do_initcall_level(level);
}
```



**因此，内核启动的时候，解析acpi-table，最后在CPU已经启动之后，加载ACPI-namespace**



##### namespace解析过程

* DSDT表

> DSDT表包含一个Definition Block，叫作'Differentiated Definition Block for the DSDT'，它包含了实现与配置信息(implementation and configuration information)。 OSPM用这些信息来实现：电源管理，热量管理，以及(在ACPI硬件寄存器所描述的信息之外的)即插即用。

解析DSDT-table: 以执行的方式解析DSDT-table ?

访问table的方式有三种分别为

```
#define ACPI_PARSE_LOAD_PASS1           0x0010
#define ACPI_PARSE_LOAD_PASS2           0x0020
#define ACPI_PARSE_EXECUTE              0x0030
#define ACPI_PARSE_MODE_MASK            0x0030
```

这里采用的是`ACPI_PARSE_EXECUTE`，为什么以执行Method的的方式加载命名空间呢？



```c
acpi_tb_load_namespace
	// acpi_gbl_root_node是全局变量，acpi namespace的根节点
	-> acpi_ns_load_table(acpi_gbl_dsdt_index, acpi_gbl_root_node);
	-> acpi_ns_load_table(i, acpi_gbl_root_node); // 遍历list中的所有table, 并设置为加载状态
		// 该函数解析table，但是不解析control methods
		// 全部的namespace全部解析完成后，才能解析control methods
		-> acpi_ns_parse_table(table_index, node);
			// 作为有个大的control method执行该table， single pass parse
			-> acpi_ns_execute_table
				-> acpi_ps_execute_table(info)
					// 下面两个函数创建并初始化 walk_state
					-> acpi_ds_create_walk_state
					-> acpi_ds_init_aml_walk
					// 解析AML， 返回ops tree
					-> acpi_ps_parse_aml(walk_state)
						-> acpi_ps_parse_loop
							// LOOP
							(-> acpi_ps_create_op
							-> acpi_ps_complete_op)
        			// 在解析完之后
        			-> acpi_ps_complete_final_op
```





### 通用： ACPI Method execute-flow



调用ACPI核心层定义函数接口为：

```c
acpi_evaluate_object // 需要提供设备路径 方法名称 和 参数
  -> acpi_ns_evaluate(info) // 执行一个控制方法
  	// 执行该方法之前，lock aml interpreter
  	// 该方法通过AML interpreter执行设备节点中定义的method
  	-> acpi_ps_execute_method(info)
  		-> acpi_ds_create_walk_state // 创建工作状态节点
  		-> acpi_ds_init_aml_walk
  		-> acpi_ps_parse_aml(walk_state)

/*
* 函数acpi_ps_parse_aml 会扫描两次，第一次扫描主要做合法性检查 并载入参数
* 第二次扫描只扫描操作节点，
* 最后将操作对象集合发布到AML分析器执行列表中，并执行
*/
```

IBM提供的一张大图：

[链接](https://www.ibm.com/developerworks/cn/linux/l-acpi/part1/index.html)

![](/img/in-post/acpi-execute-command-flow.gif)



### 通用事件处理

#### ACPI中的GPE

通过FADT获得当前的事件寄存器描述之后，就知道当前ACPI系统中事件寄存器的硬件地址，长度，各个状态位的情况，这里就有一个更加高级的数据结构介入来对每个事件（状态位）的响应调动对应的句柄，这个数据结构就是GPE Block结构：struct acpi_gpe_block_info *gpe_block，它是GPE的响应核心，在初始化ACPI驱动的阶段将会由acpi_ev_create_gpe_block这个函数创建这个数据结构,扫描名字空间（Namespace）；在名字空间中通常会有专门的全局对象节点针对GPEx_STS寄存器的各个位具体执行操作节点进行描述，如下例：_GPE用来表示GPE寄存器，以及相应的需要通知OSPM处理动作

* 一个GPE的描述示例

  ```
  _GPE
      Method(_L01) { // Update device
         Sleep(250) // Mechanical Delay 
         Notify(CDRM, 1)
      }
  ```

  **其中`\_GPE`表示当前通用寄存器的ASL表示， Method是方法节点，`L`表示电平触发，`01`表示寄存器的01位， Method表示的就是该寄存器某一位对应的寄存器的控制方法**

#### 初始化work flow

```c
acpi_enable_subsystem
	// 初始化Fixed events和GPE events
	-> acpi_ev_initialize_events()
		// 初始化 Fixed events, 设置handler=NULL, 然后disable fixed events
		-> acpi_ev_fixed_event_initialize()
		-> acpi_ev_gpe_initialize()
			-> acpi_ev_create_gpe_block
			
// acpi_ev_gpe_initialize函数中，首先获取namespace的锁， 调用函数acpi_ev_create_gpe_block，该函数根据全局变量acpi_gbl_FADT中的xgpe0_block、xgpe1_block初始化全局变量acpi_gbl_gpe_fadt_blocks数组

// 在acpi_enable_subsystem()函数的最后，初始化SCI handler和Global Lock handler
// 结束了硬件初始化流程
  -> acpi_ev_install_xrupt_handlers()
  	// 下面两个函数设置SCI中断的处理函数和global lock的处理函数
  	-> acpi_ev_install_sci_handler()   // 中断处理函数为acpi_ev_sci_xrupt_handler
  	-> acpi_ev_init_global_lock_handler()
```



一个很重要的数据结构为**struct acpi_gpe_block_info**

```c
/*
 * Information about a GPE register block, one per each installed block --
 * GPE0, GPE1, and one per each installed GPE Block Device.
 */
struct acpi_gpe_block_info {
// 当前GPE事件所需要在执行的名字空间的方法节点，如上面例中的Method(_L01)
	struct acpi_namespace_node *node;
	struct acpi_gpe_block_info *previous;
	struct acpi_gpe_block_info *next;
	struct acpi_gpe_xrupt_info *xrupt_block;	/* Backpointer to interrupt block */
// 指向当前GPE的寄存器组
	struct acpi_gpe_register_info *register_info;	/* One per GPE register pair */
	struct acpi_gpe_event_info *event_info;	/* One for each GPE */
	u64 address;		/* Base address of the block */
	u32 register_count;	/* Number of register pairs in block */
	u16 gpe_count;		/* Number of individual GPEs in block */
	u16 block_base_number;	/* Base GPE number for this block */
	u8 space_id;
	u8 initialized;		/* TRUE if this block is initialized */
};
```

数据结构`acpi_gpe_event_info`将硬件上的状态寄存器和方法节点对应起来

```c
/*
 * Information about a GPE, one per each GPE in an array.
 * NOTE: Important to keep this struct as small as possible.
 */
struct acpi_gpe_event_info {
	union acpi_gpe_dispatch_info dispatch;	/* Either Method, Handler, or notify_list */
	struct acpi_gpe_register_info *register_info;	/* Backpointer to register info */
	u8 flags;		/* Misc info about this GPE */
	u8 gpe_number;		/* This GPE */
	u8 runtime_count;	/* References to a run GPE */
	u8 disable_for_dispatch;	/* Masked during dispatching */
};

/*
 * GPE dispatch info. At any time, the GPE can have at most one type
 * of dispatch - Method, Handler, or Implicit Notify.
 */
union acpi_gpe_dispatch_info {
	struct acpi_namespace_node *method_node;	/* Method node for this GPE level */
	struct acpi_gpe_handler_info *handler;  /* Installed GPE handler */
	struct acpi_gpe_notify_info *notify_list;	/* List of _PRW devices for implicit notifies */
};
```





SCI中断的中断处理函数：

```c
acpi_ev_sci_xrupt_handler
该函数是所有SCI中断的处理函数，所有的GPE共享该中断处理句柄，
函数处理过程：
-> acpi_ev_fixed_event_detect // 该函数检查并分发fixed event
-> acpi_ev_gpe_detect(gpe_xrupt_list) // 处理并分发所有的GPE events
	-> acpi_ev_detect_gpe // 扫描acpi_gbl_gpe_xrupt_list_head全局队列上挂的GPE Block检查事件状态，。检查寄存器状态等，并分发到事件的处理进程，根据当前的使能状态寄存器决定是否响应
		-> 如果 ACPI_GPE_DISPATCH_TYPE(gpe_event_info->flags) ==
	    ACPI_GPE_DISPATCH_RAW_HANDLER，调用gpe_handler_info->address(gpe_device, gpe_number, gpe_handler_info->context); 
	  -> acpi_ev_gpe_dispatch // 该函数继续进行分发，如果gpe_event_info->flags的type为ACPI_GPE_DISPATCH_HANDLER，调用function(EC等)；如果为ACPI_GPE_DISPATCH_METHOD或者ACPI_GPE_DISPATCH_NOTIFY，就调用acpi_os_execute
	  	-> acpi_os_execute // 将要执行的函数acpi_ev_asynch_execute_gpe_method放入OS的执行队列中异步执行
					-> acpi_ev_asynch_execute_gpe_method // 该函数会执行GPE控制方法
						-> acpi_ns_evaluate(info) // 执行改GPE的控制方法， _Lxx或者_Exx
```



#### GPE dispatch method的执行

在热拔插入的处理过程的名字空间中可以看到当一个GPE事件发生之后，所调用的方法节点通常会调用"NOFITY"操作符，通知OSPM进行处理：通常一个ASL的通告操作表示如下：

`Notify(\_SB.PCI0.P2P2,0)`

\_SB.PCI0.P2P2表示当Hot Plug阶段的该设备接收响应事件，0表示ACPI向OSPM通告消息BUS Check通告, 通知PCI总线驱动层进行枚举。



常用的设备对象的OSPM通知类型：

| ACPI_NOTIFY_BUS_CHECK=0        | 总线检查：设备对象出现，通知OSPM，完成"Plug and Play"的枚举操作。当收到通知时，OSPM执行这个操作，在热拔插的时候，由ACPI AML通过Notify的方式通知该值到OSPM， |
| ------------------------------ | ------------------------------------------------------------ |
| ACPI_NOTIFY_DEVICE_CHECK=1     | 设备检查：用于通知OSPM，设备或出现或消失。如果设备出现，OSPM将从设备节点的父节点"重新枚举(re-enumerate)"。如果设备拔出，OSPM将使设备的状态定为无效。OSPM可以优化重新枚举的过程。 |
| ACPI_NOTIFY_DEVICE_WAKE=2      | 设备唤醒：用于通知OSPM设备发出了它的睡眠事件信号，OSPM需要通知OSPM的本地设备驱动，此情况仅用于支持_PRW的设备。 |
| ACPI_NOTIFY_EJECT_REQUEST=3    | 拔出要求：用以通知OSPM设备应被弹出，同时OSPM需要执行"Plug and Play 弹出操作"，与此同时OSPM将运行_Ejx方法。 |
| ACPI_NOTIFY_BUS_MODE_MISMATCH6 | 总线模式错配：用以通知OSPM设备被插入一个非当前操作模式的槽或背板中。例如：当用户试图将一个PCI设备热插入一个运行在PCI-X模式下的总线的槽中。 |
| ACPI_NOTIFY_POWER_FAULT=7      | 电源故障：用以通知OSPM，设备由于电源故障不能从D3状态下转出。 |



前面提到，SCI中断的处理函数最终会调用acpi_ev_asynch_execute_gpe_method函数，该函数会根据acpi_gpe_event_info结构体中的flags决定调用哪个dispatch函数。

如果是Notify，会进而调用函数`acpi_ev_queue_notify_request(notify->device_node, ACPI_NOTIFY_DEVICE_WAKE);`，进而调用acpi_os_execute函数，将要执行的函数`acpi_ev_notify_dispatch`放入OS执行队列中。 函数`acpi_ev_notify_dispatch`，该函数会调用global和local的handler。



* 下面看一下数据结构之间的关系

  ```c
  // acpi_generic_state是一个union
  union acpi_generic_state {
  	struct acpi_common_state common;
  	struct acpi_control_state control;
  	struct acpi_update_state update;
  	struct acpi_scope_state scope;
  	struct acpi_pscope_state parse_scope;
  	struct acpi_pkg_state pkg;
  	struct acpi_thread_state thread;
  	struct acpi_result_values results;
  	struct acpi_notify_info notify;  // NOtify相关
  };
  
  /*
   * Notify info - used to pass info to the deferred notify
   * handler/dispatcher.
   */
  struct acpi_notify_info {
  	ACPI_STATE_COMMON u8 handler_list_id;
  	struct acpi_namespace_node *node;   // 方法节点
  	union acpi_operand_object *handler_list_head;
  	struct acpi_global_notify_handler *global;  // global handler
  };
  
  // acpi_operand_object是一个union, 所有的object类型
  // 其中的Notify object 为acpi_object_notify_handler
  
  struct acpi_object_notify_handler {
  	ACPI_OBJECT_COMMON_HEADER struct acpi_namespace_node *node;	/* Parent device */
  	u32 handler_type;	/* Type: Device/System/Both */  // 类型：系统消息通告和设备消息通告
  	acpi_notify_handler handler;	/* Handler address */  // 处理函数
  	void *context;
  	union acpi_operand_object *next[2];	/* Device and System handler lists */
  };
  ```

* Notify-handler是何时注册的？

  ACPI提供了对外的接口函数`acpi_notify_handler`， kernel内的驱动程序可以调用该接口注册事件处理函数。

  `acpi_install_notify_handler(acpi_handle device, u32 handler_type, acpi_notify_handler handler, void *context)`函数参数说明： device表示接收通告的设备句柄，type表示为系统通告或者设备通告类型，handler表示回调函数，context为回调函数的参数



### ASL & AML 

* ASL(ACPI Source Language)：ASL在经过编译器编译后，变成AML(ACPI Machine Language)。
* AML(ACPI Machine Language)： 是一种BYTECODE， 由OSPM执行。



#### 设备标识

##### 硬件 ID（_HID）

在 ACPI 中标识设备的最低要求是硬件 ID （_HID）对象。供应商 Id 在整个行业中必须是唯一的。 Microsoft 会分配这些字符串以确保它们是唯一的。 可以从[即插即用 ID-PNPID 请求](https://go.microsoft.com/fwlink/p/?linkid=330999)中获取供应商 id。

**注意** ACPI 5.0 还支持在 _HID 和其他标识对象中使用 PCI 分配的供应商 id，因此你可能不需要从 Microsoft 获取供应商 ID。 有关硬件标识要求的详细信息，请参阅[ACPI 5.0 规范](https://uefi.org/specifications)的 "_HID （硬件 ID）" 部分。



##### 兼容ID (_CID)

对于与 Windows 附带的收件箱驱动程序兼容的设备，Microsoft 保留了供应商 ID "PNP"。 Windows 定义了多个与此供应商 ID 结合使用的设备 Id，该 ID 可用于为设备加载 Windows 提供的驱动程序。 兼容 ID （_CID）对象是单独的对象，用于返回这些标识符。 Windows 始终优先于 INF 匹配和驱动程序选择中的兼容 Id （由 _CID 返回）上的硬件 Id （由 _HID 返回）。 如果供应商提供的特定于设备的驱动程序不可用，则此首选项允许将 Windows 提供的驱动程序视为默认驱动程序。 



##### 子系统 ID （\_SUB）、硬件修订版本（\_HRV）和类（\_CLS）

OEM 系统上的设备 Id 是 "四部分" Id。 这四个部分分别是供应商 ID、设备 ID、子系统供应商（OEM） ID 和子系统（OEM）设备 ID。 因此，对于 OEM 平台，需要子系统 ID （_SUB）对象。

##### Address （\_ADR）唯一ID （\_UID）

设备标识有三个附加要求：

- 对于连接到硬件可枚举的父总线（例如，SDIO、USB HSIC），但具有平台特定功能或控件（例如，sideband 电源或唤醒中断）的设备，不使用 \_HID。 相反，设备标识符由父总线驱动程序创建（如前文所述）。 但在这种情况下，地址对象（\_ADR）需要位于设备的 ACPI 命名空间中。 此对象使操作系统能够将总线枚举设备与 ACPI 描述的功能或控件相关联。
- 在使用特定 IP 块的多个实例的平台上，因此每个块都具有相同的设备标识对象，这是唯一标识符（_UID）对象，使操作系统能够区分块。
- 特定命名空间范围中的两个设备不能具有相同的名称。



#### 资源配置

对于命名空间中标识的每个设备，**当前资源设置（\_CRS）对象还必须报告设备使用的系统资源（内存地址、中断等）**。 支持对多个可能的资源配置（_PR）和用于更改设备资源配置（_SRS）的控件进行报告，但这是可选的。

用于 SoC 平台的新的是设备可以使用的 GPIO 和简单外围总线（SPB）资源。 有关详细信息，请参阅[常规用途 i/o （GPIO）](https://docs.microsoft.com/zh-cn/windows-hardware/drivers/bringup/general-purpose-i-o--gpio-)和[简单外围总线（SPB）](https://docs.microsoft.com/zh-cn/windows-hardware/drivers/bringup/simple-peripheral-bus--spb-)。

另外，对于 SoC 平台，还可以使用常规用途固定 DMA 描述符。 FixedDMA 描述符支持多个系统设备共享 DMA 控制器硬件。 静态分配给特定系统设备的 DMA 资源（请求行和通道寄存器）在 FixedDMA 描述符中列出。 有关详细信息，请参阅[ACPI 5.0 规范](https://uefi.org/specifications)的 "FIXEDDMA （DMA 资源描述符宏）" 19.5.49 部分。

##### 设备状态更改

出于多种原因，可以禁用或删除 ACPI 枚举设备。 **提供状态（_STA）对象是为了使此类状态更改能够传递到操作系统**。 有关 _STA 的说明，请参阅[ACPI 5.0 规范](https://uefi.org/specifications)的6.3.7 部分。 Windows 使用 _STA 来确定是否应枚举设备，并将其显示为已禁用或对用户不可见。 此控制在固件中适用于许多应用程序，包括停靠和 USB OTG 主机到功能切换。

此外，**ACPI 还提供了一种通知机制，ASL 可以使用该机制来通知平台中事件的驱动程序，如作为插接的一部分被删除的设备。 通常，当 ACPI 设备的状态发生更改时，固件必须执行 "设备检查" 或 "总线检查" 通知，以使 Windows 重新枚举设备并重新评估其 _STA**。 有关 ACPI 通知的信息，请参阅[acpi 5.0 规范](https://uefi.org/specifications)的 "设备对象通知" 部分5.6.6。











### PCI with ACPI 

work flow：

```c
// arch_initcall(acpi_pci_init); 因此该函数也是在start_kernel函数最后do_initcalls()中被调用

/* just register struct acpi_bus_type acpi_pci_bus to list: bus_type_list */
acpi_pci_init() /* arch_initcall(acpi_pci_init) */
 
acpi_init() /* subsys_initcall(acpi_init) */
    --> acpi_scan_init()
        /*
         * register pci_root_handler to list: acpi_scan_handlers_list.
         * the attach will be called in acpi_scan_attach_handler().
         * there attach is assigned as acpi_pci_root_add()
         */
        --> acpi_pci_root_init()
        /*
         * register pci_link_handler to list: acpi_scan_handlers_list.
         * this handler has relationship with PCI IRQ.
         */
        --> acpi_pci_link_init()
        /* we facus on PCI-ACPI, ignore other handlers' init */
        ...
  			/* 在root scope下查找，找到所有的设备，首先添加根设备，
  			 * 然后acpi_walk_namespace,添加scope下的所有节点
  			 */
        --> acpi_bus_scan(ACPI_ROOT_OBJECT)
  					-> acpi_bus_check_add(handle, 0, NULL, &device)
            /* create struct acpi_devices for all device in this system */
            --> acpi_walk_namespace(acpi_bus_check_add)
            --> acpi_bus_attach()
                --> acpi_scan_attach_handler() 
                    --> acpi_scan_match_handler()
                    --> handler->attach /* attach is acpi_pci_root_add */
 
acpi_pci_root_add()
    /*
     * in kernel, there are two pci_acpi_scan_root, they are in
     * arch/ia64/pci/pci.c and arch/x86/pci/acpi.c.
     * if we will implement PCI using ACPI in ARM64, we should implement
     * another this kind of function in arch/arm64/kernel/pci.c.
     * in pci_acpi_scan_root, will allocate struct pci_controller and
     * struct pci_root_info.
     */
    --> pci_acpi_scan_root()
        --> probe_pci_root_info()
	    /*
	     * will called twice, first for count_window, second for add window.
	     * this function will get infomation from ACPI table.
	     */
             --> acpi_walk_resources() /* drivers/acpi/acpica/rsxface.c */
```



b. basic structs:

```c
global list: acpi_scan_handlers_list /* drivers/acpi/scan.c */
element: struct acpi_scan_handler

static list: bus_type_list /* drivers/acpi */
element: struct acpi_bus_type

struct acpi_scan_handler:
     struct acpi_device_id *ids;
     attach;
     ...

struct acpi_bus_type
struct acpi_device  // 在APCI层上描述一个设备

struct pci_controller
struct pci_root_info:
    struct pci_controller *controller;
```



c. initialization

在PCIe hotplug driver中， 驱动加载函数

```c
pciehp_probe
	-> pcie_init_notification
		-> pciehp_request_irq
			-> irq = ctrl->pcie->irq;
			-> request_threaded_irq(irq, pciehp_isr, pciehp_ist,
				      IRQF_SHARED, "pciehp", ctrl);

// PCIe对应的中断处理函数为pciehp_ist
pciehp_ist
	-> pciehp_handle_presence_or_link_change
		-> pciehp_enable_slot
			-> __pciehp_enable_slot
				-> board_added(ctrl)
					-> cpqhp_configure_device(ctrl, new_slot);
						-> pci_hp_add_bridge(func->pci_dev);
							-> pci_scan_bridge_extend(parent, dev, busnr, available_buses, 1);
								-> pci_add_new_bus
									-> pci_alloc_child_bus
										-> pcibios_add_bus(child)
											-> acpi_pci_add_bus
												-> acpi_pci_slot_enumerate
													-> acpi_walk_namespace(ACPI_TYPE_DEVICE, handle, 1,
				    register_slot, NULL, bus, NULL);

// 函数 register_slot： 如果handle有对应的slot, 将设备挂载到对应的插槽结构中，如果没有对应的插槽，则创建

//对应的PCI插槽结构
/* pci_slot represents a physical slot */
struct pci_slot {
	struct pci_bus		*bus;		/* Bus this slot is on */
	struct list_head	list;		/* Node in list of slots */
	struct hotplug_slot	*hotplug;	/* Hotplug info (move here) */
	unsigned char		number;		/* PCI_SLOT(pci_dev->devfn) */
	struct kobject		kobj;
};

```



