---
layout:     post
title:      "virtio-pci-device summary"
subtitle:   "  "
date:       2020-01-14 
author:     "Maggie"
header-img: "img/post-bg-digital-native.jpg"
catalog: true
tags:
    - pci
    - virtio
---



### PCI设备

#### PCI配置空间

* header-type=00h

  ![](/img/in-post/post-pci-device2.bmp)

* header-type=01h(pci-to-pci-bridge)

  ![](/img/in-post/post-pci-device2.bmp)



#### PCI配置空间的访问方式

- IO方式： CONFIG_ADDRESS=0xcf8， CONFIG_DATA=0xcfc，分别为地址和数据

  | 31         | 30 - 24  | 23 - 16    | 15 - 11       | 10 - 8          | 7 - 0            |
  | ---------- | -------- | ---------- | ------------- | --------------- | ---------------- |
  | Enable Bit | Reserved | Bus Number | Device Number | Function Number | Register Offset¹ |

  当CPU想访问一个设备的配置空间时，生成地址，写入IO端口，bus根据收到的地址，找到相应的设备，修改设备的config

- 第二种方式在spec2.0中已经去掉



#### bar空间

MMIO类型

| 31 - 4                       | 3            | 2 - 1 | 0        |
| ---------------------------- | ------------ | ----- | -------- |
| 16-Byte Aligned Base Address | Prefetchable | Type  | Always 0 |

IO 类型

| 31 - 2                      | 1        | 0        |
| --------------------------- | -------- | -------- |
| 4-Byte Aligned Base Address | Reserved | Always 1 |

bar空间的地址存放在PCI的configuration中，范围为0x10~0x27.

* 换一个角度， bar可以分为64位和32位：如果为64位，则一个设备最多有3个bar，bar空间的地址可以是1~64GB的范围。如果为32位，一个设备可以有6个bar，bar空间的地址必须在memory-hole（3.5G~4G）内
* 此外，还有rom类型的bar



#### PCI 设备发现和初始化

* 首先看一下得到bar空间大小的方式

  ​	**a.**向BAR寄存器写全1

  ​	**b.**读回寄存器里面的值，然后clear 上图中特殊编码的值，(IO 中bit0，bit1， memory中bit0-3)。

  ​	**c.**对读回来的值去反，加一就得到了该设备需要占用的地址内存空间。

* PCI 设备枚举
  * 深度优先遍历，遍历整个设备树。如果是bus，则遍历设备子树
  * 在遍历的过程中，通过写读设备的bar-reg，分配bar空间
  * 设置必要的cap

* 寻找设备和访问设备配置空间

  * 正向译码：通过地址寻找设备，读写配置空间
  * 负向译码

  >在PCI总线中定义了两种“地址译码”方式，一个是正向译码，一个是负向译码。当访问Bus N时，其下的所有PCI设备都将对出现在地址周期中的PCI总线地址进行译码。如果这个地址在某个PCI设备的BAR空间中命中时，这个PCI设备将接收这个PCI总线请求。这个过程也被称为PCI总线的正向译码，这种方式也是大多数PCI设备所采用的译码方式。
  >
  >但是在PCI总线上的某些设备，如PCI-to-(E)ISA桥（或LPC）并不使用正向译码接收来自PCI总线的请求， PCI BUS N上的总线事务在三个时钟周期后，没有得到任何PCI设备响应时(即总线请求的PCI总线地址不在这些设备的BAR空间中)，PCI-to-ISA桥将被动地接收这个数据请求。这个过程被称为PCI总线的负向译码。可以进行负向译码的设备也被称为负向译码设备。
  >
  >在PCI总线中，除了PCI-to-(E)ISA桥可以作为负向译码设备，PCI桥也可以作为负向译码设备，但是PCI桥并不是在任何时候都可以作为负向译码设备。在绝大多数情况下，PCI桥无论是处理“来自上游总线(upstream)”，还是处理“来自下游总线（downstream)”的总线事务时，都使用正向译码方式。



#### PCI-Capability

* 每个PCI-Calability都有一个CapabilityID，内核在加载设备驱动时，会读取设备的Capability

* 中断相关的Capability
  * Msix对应的CapabilityID=0x11, MSI对应的ID为 0x5

  * MSI直接将Massage-Address和Message-Data放在Cap中

  * MSIX-Cap存放的是MSIx-table和pending-table存放的bar-index和offset，一般msix-table和pending-table存放在同一个bar中





### virtio-pci设备

#### virtio-pci设备的bar空间



- 旧的virtio协议采用 lagacy模式，将virtio设备相关的配置放在bar-0中

- 新的virtio协议modern 模式中的配置结构共有五种，分别为

  ```c
  /* Common configuration */ 
  #define VIRTIO_PCI_CAP_COMMON_CFG        1 
  /* Notifications */ 
  #define VIRTIO_PCI_CAP_NOTIFY_CFG        2
  /* ISR Status */ 
  #define VIRTIO_PCI_CAP_ISR_CFG           3 
  /* Device specific configuration */ 
  #define VIRTIO_PCI_CAP_DEVICE_CFG        4 
  /* PCI configuration access */ 
  #define VIRTIO_PCI_CAP_PCI_CFG           5 
  ```

  5种配置结构都存放在设备的bar空间中，可以有不同的偏移。可以通过结构体`struct virtio_pci_cap `中指定的bar-index和offset找到。

  

  * 结构体`struct virtio_pci_cap `如下，cft_type为上述的五种类型

    ```c
    struct virtio_pci_cap {
    	u8 cap_vndr; /* Generic PCI field: PCI_CAP_ID_VNDR = 0x9 */
    	u8 cap_next; /* Generic PCI field: next ptr. */
    	u8 cap_len; /* Generic PCI field: capability length */
    	u8 cfg_type; /* Identifies the structure. */
    	u8 bar; /* Where to find it. */
    	u8 padding[3]; /* Pad to full dword. */
    	le32 offset; /* Offset within bar. */
    	le32 length; /* Length of the structure, in bytes. */
    };
    ```

    

  * 在qemu中，为了向前兼容，bar-0依旧用作存放legacy模式下的配置信息。

    * notify_io config                                modern-io       MR       bar-2
    * notify/isr/device/ common cfg      modern-mem MR       bar-4
    * msix-table/pending-table                                                      bar-1                

  * cloud-hypervisor只支持medern模式，几种config对应bar-0中不同的偏移，cloud-hypervisor中定义的偏移分别为

    ```rust
    const COMMON_CONFIG_BAR_OFFSET: u64 = 0x0000;
    const COMMON_CONFIG_SIZE: u64 = 56;
    const ISR_CONFIG_BAR_OFFSET: u64 = 0x2000;
    const ISR_CONFIG_SIZE: u64 = 1;
    const DEVICE_CONFIG_BAR_OFFSET: u64 = 0x4000;
    const DEVICE_CONFIG_SIZE: u64 = 0x1000;
    const NOTIFICATION_BAR_OFFSET: u64 = 0x6000;
    const NOTIFICATION_SIZE: u64 = 0x1000;
    const MSIX_TABLE_BAR_OFFSET: u64 = 0x8000;
    const MSIX_TABLE_SIZE: u64 = 0x40000;
    const MSIX_PBA_BAR_OFFSET: u64 = 0x48000;
    // The size is 2KiB because the Pending Bit Array has one bit per vector and it
    // can support up to 2048 vectors.
    const MSIX_PBA_SIZE: u64 = 0x800;
    //bar-0 的大小
    const CAPABILITY_BAR_SIZE: u64 = 0x80000;
    ```

**cloud-hypervisor的初始化过程：**

- bar-0初始化结束之后，会将bar-0的地址（在cloud-hypervisor中，地址是分配的）设置到pci configuration中

- 然后，设置virtio设备相关的配置结构： 指定在bar-0中的偏移和大小，即（cfg-type, bar-index, offset, size）

  并将该配置，设置到PCI configuration中的capability-list中

> 观察read_bar()函数，会根据offset属于哪个配置结构，决定操作设备的哪个cfg

> 内核驱动加载时，除非强制选择legacy模式， 否则都选择modern模式

#### virtio-pci设备中断

##### queue-interrupt

* 驱动程序写config，选择当前需要设置的virtqueue
* 设置该queue的size和DescTable
* 如果存在msix-Capability并且已经enable, 选择一个vector, 并设置到queue_msix_vector字段(msix-table的entry-idx)中。该vector指定queue-event触发的中断

##### config-change-interrupt

* config_msix_vector和queue_msix_vector都是common_cfg中的字段，用于设置config和queues对应的vector

```rust
write(BusDevice)
  -> write_bar(VirtioPciDevice)
	-> self.common_config.write()
/// * Registers:
/// ** About the whole device.
/// le32 device_feature_select;     // 0x00 // read-write
/// le32 device_feature;            // 0x04 // read-only for driver
/// le32 driver_feature_select;     // 0x08 // read-write
/// le32 driver_feature;            // 0x0C // read-write
/// le16 msix_config;               // 0x10 // read-write
/// le16 num_queues;                // 0x12 // read-only for driver
/// u8 device_status;               // 0x14 // read-write (driver_status)
/// u8 config_generation;           // 0x15 // read-only for driver
/// ** About a specific virtqueue.
/// le16 queue_select;              // 0x16 // read-write
/// le16 queue_size;                // 0x18 // read-write, power of 2, or 0.
/// le16 queue_msix_vector;         // 0x1A // read-write
/// le16 queue_enable;              // 0x1C // read-write (Ready)
/// le16 queue_notify_off;          // 0x1E // read-only for driver
/// le64 queue_desc;                // 0x20 // read-write
/// le64 queue_avail;               // 0x28 // read-write
/// le64 queue_used;                // 0x30 // read-write
```



##### 中断处理过程

* msix-table和pending-table保存在bar空间中

* 通过向msix-table中的Message Address中写Message Data，触发中断

  ![](D:\temp\Maggie262.github.io\_posts\2020-01-14-virtio-pci-device-initialization.assets\post-pci-device-msix-table.bmp)



* virtio-PCI 设备的中断处理过程
  * 如果设备需要发中断，首先判断是queue-event还是config-change-event，设置vector为相应的msix-table的entry, 找到table中的Message Address和Message Data, 发中断
  * 在发中断之前，检查msix-table的entry对应的vendor control register是否为1，如果为1.设置entry对应的pending位为1，先不发中断。当CPU清除该msix-table entry的vendor control register， 会触发msix中断发送，并将pending位置0 

```
// cloud-hypervisor
add_virtio_pci_device
	-> 定义msix_cb，即发中断
	-> virtio_pci_device.assign_msix(msi_cb);
		-> 进一步定义VirtioInterrupt的回调函数，会根据参数中的InterruptType触发config-change-interrupt或者queue-interrupt, 调用msix_cb，发中断

// 后面在设备激活的时候会传入该回调函数
// 设备的virtqueue事件处理完成后，触发该回调函数
```



[https://example61560.wordpress.com/2016/06/30/pcipcie-%E6%80%BB%E7%BA%BF%E6%A6%82%E8%BF%B06/](https://example61560.wordpress.com/2016/06/30/pcipcie-总线概述6/)







### msix

#### 数据结构



内核中与中断相关的数据结构：

```c
struct msi_msg {
>---u32>address_lo;>/* low 32 bits of msi message address */
>---u32>address_hi;>/* high 32 bits of msi message address */
>---u32>data;>-->---/* 16 bits of msi message data */
};

struct msi_desc {
>---/* Shared device/bus type independent data */
>---struct list_head>--->---list;
>---unsigned int>--->--->---irq;
>---unsigned int>--->--->---nvec_used;
>---struct device>-->--->---*dev;   // 关联设备
>---struct msi_msg>->--->---msg;
>---struct cpumask>->--->---*affinity;

>---union {
>--->---/* PCI MSI/X specific data */
>--->---struct {
>--->--->---u32 masked;
>--->--->---struct {
>--->--->--->---__u8>---is_msix>>---: 1;
>--->--->--->---__u8>---multiple>---: 3;
>--->--->--->---__u8>---multi_cap>--: 3;
>--->--->--->---__u8>---maskbit>>---: 1;
>--->--->--->---__u8>---is_64>-->---: 1;
>--->--->--->---__u16>--entry_nr;
>--->--->--->---unsigned default_irq;
>--->--->---} msi_attrib;
>--->--->---union {
>--->--->--->---u8>-mask_pos;
>--->--->--->---void __iomem *mask_base;
>--->--->---};
>--->---};

>--->---/*
>--->--- * Non PCI variants add their data structure here. New
>--->--- * entries need to use a named structure. We want
>--->--- * proper name spaces for this. The PCI part is
>--->--- * anonymous for now as it would require an immediate
>--->--- * tree wide cleanup.
>--->--- */
>--->---struct platform_msi_desc platform;
>--->---struct fsl_mc_msi_desc fsl_mc;
>---};
};

/**
 * struct irq_common_data - per irq data shared by all irqchips
 * @state_use_accessors: status information for irq chip functions.
 *>->--->---Use accessor functions to deal with it
 * @node:>-->---node index useful for balancing
 * @handler_data:>--per-IRQ data for the irq_chip methods
 * @affinity:>-->---IRQ affinity on SMP. If this is an IPI
 *>->--->---related irq, then this is the mask of the
 *>->--->---CPUs to which an IPI can be sent.
 * @effective_affinity:>The effective IRQ affinity on SMP as some irq
 *>->--->---chips do not allow multi CPU destinations.
 *>->--->---A subset of @affinity.
 * @msi_desc:>-->---MSI descriptor
 * @ipi_offset:>>---Offset of first IPI target cpu in @affinity. Optional.
 */
struct irq_common_data {
>---unsigned int>--->---__private state_use_accessors;
#ifdef CONFIG_NUMA
>---unsigned int>--->---node;
#endif
>---void>--->--->---*handler_data;
>---struct msi_desc>>---*msi_desc; // msi descriptor
>---cpumask_var_t>-->---affinity;
#ifdef CONFIG_GENERIC_IRQ_EFFECTIVE_AFF_MASK
>---cpumask_var_t>-->---effective_affinity;
#endif
#ifdef CONFIG_GENERIC_IRQ_IPI
>---unsigned int>--->---ipi_offset;
#endif
};

/**
 * struct irq_data - per irq chip data passed down to chip functions
 * @mask:>-->---precomputed bitmask for accessing the chip registers
 * @irq:>--->---interrupt number
 * @hwirq:>->---hardware interrupt number, local to the interrupt domain
 * @common:>>---point to data shared by all irqchips
 * @chip:>-->---low level interrupt hardware access
 * @domain:>>---Interrupt translation domain; responsible for mapping
 *>->--->---between hwirq number and linux irq number.
 * @parent_data:>---pointer to parent struct irq_data to support hierarchy
 *>->--->---irq_domain
 * @chip_data:>->---platform-specific per-chip private data for the chip
 *>->--->---methods, to allow shared chip implementations
 */
struct irq_data {
>---u32>>--->---mask;
>---unsigned int>--->---irq;
>---unsigned long>-->---hwirq;
>---struct irq_common_data>-*common;
>---struct irq_chip>>---*chip;
>---struct irq_domain>--*domain;
#ifdef>-CONFIG_IRQ_DOMAIN_HIERARCHY
>---struct irq_data>>---*parent_data;  // 保存父级 
#endif
>---void>--->--->---*chip_data; // chip_data = apic_chip_data
};

struct apic_chip_data {
>---struct irq_cfg>->---hw_irq_cfg;
>---unsigned int>--->---vector;
>---unsigned int>--->---prev_vector;
>---unsigned int>--->---cpu;
>---unsigned int>--->---prev_cpu;
>---unsigned int>--->---irq;
>---struct hlist_node>--clist;
>---unsigned int>--->---move_in_progress>---: 1,
>--->--->--->---is_managed>->---: 1,
>--->--->--->---can_reserve>>---: 1,
>--->--->--->---has_reserved>--->---: 1;
};

```



 



#### msix初始化流程

Message Data中的vector设置： 内核申请irq号 和vector，设置per_cpu变量中的vector_irq数组（下标是vector，保存的irq号）， 最后设置PCI设备的配置区间，即Message Address和Message Data（vector号为Data的后8位）。

```
vp_request_msix_vectors
	-> pci_alloc_irq_vectors_affinity 
		-> __pci_enable_msix_range
			-> __pci_enable_msix
				-> msix_capability_init
					-> pci_msi_setup_msi_irqs
						-> msi_domain_alloc_irqs (or arch_setup_msi_irqs)
	// 将irq号与handler关联
	-> request_irq(pci_irq_vector(vp_dev->pci_dev, v), handler, ...)
		-> request_threaded_irq
			-> __setup_irq(irq, desc, action)
```



```c
//如果是arch_setup_msi_irqs
arch_setup_msi_irqs
	-> native_setup_msi_irqs
		-> domain = msi_default_domain = pci_msi_create_irq_domain(fn, &pci_msi_domain_info, parent)
			-> msi_domain_alloc_irqs(domain, &dev->dev, nvec);
				-> (every_entry) __irq_domain_alloc_irqs
					|-> irq_domain_alloc_descs  //分配irq号并初始化所有相关的数据结构
						-> __irq_alloc_descs
							// 其中全局变量allocated_irqs为bitmap，表示irq号是否已经分配
							-> bitmap_find_next_zero_area(allocated_irqs, IRQ_BITMAP_BITS,
							-> alloc_descs
								-> alloc_desc // 初始化irq_desc
								-> irq_insert_desc
								// // 设置bitmap的相应位， 表示已分配
								-> bitmap_set(allocated_irqs, start, cnt) 
					|-> irq_domain_alloc_irqs_hierarchy
						-> domain->ops->alloc(domain, irq_base, nr_irqs, arg);
							->  = x86_vector_alloc_irqs
								-> assign_irq_vector_policy
									-> assign_irq_vector
										-> assign_vector_locked
											// 分配irq对应的vector
											// 设置apic_chip_data中的vector字段
											-> irq_matrix_alloc
											// 更新irq对应的apic_data变量，并设置per_cpu变量
											-> apic_update_vector
											// 更新irq对应的apci_data变量中的irq_cfg变量
											-> apic_update_vector_cfg
                 //设置 desc->irq_common_data.msi_desc
                 // 设置 desc->irq = virq
                 -> (every_entry in desc->nvec_used) irq_set_msi_desc_off(virq, i, msi_desc);
				 -> (for_each_msi_entry) irq_domain_activate_irq
					-> __irq_domain_activate_irq
						-> domain->ops->activate(domain, irqd, reserve)
                        	-> msi_domain_activate
                        		// (构造Message Address和Message Data的流程)
                        		-> irq_chip_compose_msi_msg
                        			// 递归找到最顶级的irq_data
                        			-> pos->chip->irq_compose_msi_msg(pos, msg);
                                        // 设置 Message Address和data的internal函数
                                    	-> irq_msi_compose_msg
                        		// 写配置空间
                        		-> irq_chip_write_msi_msg
                        			-> chip->irq_write_msi_msg(data, msg);
                                    	-> pci_msi_domain_write_msg
                                    		// 将desc中的Message Address和Message Data写到
                                    		// PCI设备的配置空间中
                                    		-> __pci_write_msi_msg 
                                                          
/*
static const struct irq_domain_ops x86_vector_domain_ops = {
>---.alloc>->---= x86_vector_alloc_irqs,
>---.free>-->---= x86_vector_free_irqs,
>---.activate>--= x86_vector_activate,
>---.deactivate>= x86_vector_deactivate,
#ifdef CONFIG_GENERIC_IRQ_DEBUGFS
>---.debug_show>= x86_vector_debug_show,
#endif
};
*/
                                                          
start_kernel
	-> early_irq_init
		-> arch_early_irq_init
			-> irq_set_default_host(x86_vector_domain);
            	-> irq_default_domain = x86_vector_domain_ops
           	-> arch_init_msi_domain(x86_vector_domain);

```

Message Address和Message Data是怎么构成的？

```c
static void irq_msi_compose_msg(struct irq_data *data, struct msi_msg *msg)
{
    // 由irq_data找到irq_cfg
>---struct irq_cfg *cfg = irqd_cfg(data);

>---msg->address_hi = MSI_ADDR_BASE_HI;

>---if (x2apic_enabled())
>--->---msg->address_hi |= MSI_ADDR_EXT_DEST_ID(cfg->dest_apicid);

>---msg->address_lo =
>--->---MSI_ADDR_BASE_LO |
>--->---((apic->irq_dest_mode == 0) ?
>--->--->---MSI_ADDR_DEST_MODE_PHYSICAL :
>--->--->---MSI_ADDR_DEST_MODE_LOGICAL) |
>--->---MSI_ADDR_REDIRECTION_CPU |
>--->---MSI_ADDR_DEST_ID(cfg->dest_apicid);

>---msg->data =
>--->---MSI_DATA_TRIGGER_EDGE |
>--->---MSI_DATA_LEVEL_ASSERT |
>--->---MSI_DATA_DELIVERY_FIXED |
>--->---MSI_DATA_VECTOR(cfg->vector);  ?/ 设置最低8位为vector
}

// arch/x86/include/asm/msidef.h
#define MSI_ADDR_BASE_HI>--->---0
#define MSI_ADDR_BASE_LO>--->---0xfee00000

```



* 中断属性的修改

在用户通过echo xxx > /proc/irq/xxx/affinity来调整中断的绑定属性时，内核会重新为该中断分配一个新的在对应核心上可用的vector，但是irq号不会改变。绑定属性调整的调用路径大致为irq_affinity_proc_fops===>irq_affinity_proc_write===> write_irq_affinity===>irq_set_affinity===>__irq_set_affinity_locked===>chip->irq_set_affinity(msi_set_affinity)。也就是最终通过msi_set_affinity来实现，在该函数中首先通过 __ioapic_set_affinity在绑定属性要求的cpu中选择空闲vector，然后通过__write_msi_msg把配置写入PCIE配置区。

```
arch_init_msi_domain
	-> msi_default_domain = pci_msi_create_irq_domain(fn, &pci_msi_domain_info, parent)
    	-> pci_msi_domain_update_chip_ops(info);
			-> chip->irq_write_msi_msg = pci_msi_domain_write_msg;

// 下面是msix--irq分配完成后，调用函数将msg相关信息写到PCI设备的配置空间
pci_msi_domain_write_msg
	-> __pci_write_msi_msg(struct msi_desc *entry, struct msi_msg *msg)
	// write_config, 将msg中的 Message Address 和 Message Data 写到 设备的msix相关配置结构中
```



https://wiki.osdev.org/PCI#Configuration_Space_Access_Mechanism_.231

[https://example61560.wordpress.com/2016/06/30/pcipcie-%E6%80%BB%E7%BA%BF%E6%A6%82%E8%BF%B06/](https://example61560.wordpress.com/2016/06/30/pcipcie-总线概述6/)

https://cloud.tencent.com/developer/article/1517862

http://lihanlu.cn/x86-intr-1/









