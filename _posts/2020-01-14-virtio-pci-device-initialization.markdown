





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



#### PCI 设备初始化

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



#### virtio-pci设备

**virtio设备只使用bar-0**

初始化bar空间： 

- bar-0为IO类型，用于存放virtio-pci设备不同的config

  virtio协议中的配置结构共有五种，分别为

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

  几种config对应bar-0中不同的偏移，cloud-hypervisor中定义的偏移分别为

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
  //bar-0 的大小
  const CAPABILITY_BAR_SIZE: u64 = 0x80000;
  ```

  

- bar-0初始化结束之后，会将bar-0的地址（在cloud-hypervisor中，地址是分配的）设置到pci configuration中

- 然后，设置virtio设备相关的配置结构： 指定在bar-0中的偏移和大小，即（cfg-type, bar-index, offset, size）

  并将该配置，设置到PCI configuration中的capability-list中

> 观察read_bar()函数，会根据offset属于哪个配置结构，决定操作设备的哪个cfg





