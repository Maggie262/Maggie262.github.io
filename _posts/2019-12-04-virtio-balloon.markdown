---
layout:     post
title:      "virtio-balloon分析"
subtitle:   "  "
date:       2019-12-04 
author:     "yang"
header-img: "img/post-bg-digital-native.jpg"
catalog: true
tags:
    - memory
    - virtio
---

### qemu-virtio balloon



#### 数据结构

```c
typedef struct VirtIOBalloon {
    VirtIODevice parent_obj;
    VirtQueue *ivq, *dvq, *svq;  // 3个 virt queue
    // pages we want guest to give up 
   	uint32_t num_pages; 
    // pages in balloon
    uint32_t actual;
    uint64_t stats[VIRTIO_BALLOON_S_NR];  // status 
    
    // status virtqueue 会用到
    VirtQueueElement *stats_vq_elem;
    size_t stats_vq_offset;
    
    // 定时器, 定时查询功能
    QEMUTimer *stats_timer;
    int64_t stats_last_update;
    int64_t stats_poll_interval;
    
    // features
    uint32_t host_features;
} VirtIOBalloon;
```

分析：

* `num_pages`字段是balloon表示 我们希望guest归还给host的内存大小
  * 更新： qmp命令->回调函数， 更新balloon的num_pages字段。然后通过中断注入通知guest
* `actual`字段表示balloon实际捕获的pages数目
  * guest处理configuration change中断，完成之后调用`virtio_cwrite`函数。因为写balloon设备的配置空间，所以陷出，qemu收到后会找到balloon设备，修改config
  * 修改config时，更新balloon->actual字段	

* `stats_last_update`在每次从status virtioqueue中取出数据时更新



#### 初始化



* **instance_init**

  ```c
  virtio_balloon_instance_init
  	// guest-stats,获取guest的status?????
  	-> balloon_stats_get_all
  	// 获取和设置interval， 定时查询
  	-> balloon_stats_get_poll_interval
  	-> balloon_stats_set_poll_interval
  ```

  

  ```c
  static void virtio_balloon_instance_init(Object *obj)
  {
      VirtIOBalloon *s = VIRTIO_BALLOON(obj);
  
      object_property_add(obj, "guest-stats", "guest statistics",
                          balloon_stats_get_all, NULL, NULL, s, NULL);
  
      object_property_add(obj, "guest-stats-polling-interval", "int",
                          balloon_stats_get_poll_interval,
                          balloon_stats_set_poll_interval,
                          NULL, s, NULL);
  }
  ```

  

* **realize函数**

  ```c
  virtio_balloon_device_realize
  	-> qemu_add_balloon_handler(virtio_balloon_to_target, virtio_balloon_stat)
  	
  	// 三个virtio-queue的回调函数
  	s->ivq = virtio_add_queue(vdev, 128, virtio_balloon_handle_output);
    	s->dvq = virtio_add_queue(vdev, 128, virtio_balloon_handle_output);
      s->svq = virtio_add_queue(vdev, 128, virtio_balloon_receive_stats);
      
      // 初始化之后调用
      reset_status()
  ```



#### 几个回调函数

* **inflate/deflate virt-queue的回调函数**

  virtio_balloon_handle_output

  ```c
  // 获取ram地址， 找到对应的memory region, 
  // 从该地址处balloon-page  （根据vq的不同，判断标记page的类型）
  // 放入vring，通知前端
  
  tips: 
  // 每次只balloon一个page出来
  // 将一个page标记为WILL_NEED或者DONTNEED
  ```

  

* **status virt-queue的回调函数**

  ```c
  virtio_balloon_receive_stats()
  
  // 从vring中取出元素
  更新： 
  	s->stats_vq_elem
  	s->stats_vq_offset
  	s->stats_last_update
  	s->stats
  	
  // 更新定时器
  	balloon_stats_change_timer(s, s->stats_poll_interval);
  	// 当前时刻 + interval之触发回调函数
  	// 该回调函数会将vb中的status字段放入queue中，通知前端
  ```



#### **qmp的回调函数**

```c
virtio_balloon_to_target

// target > ram_size:   target <-- ram_size
更新：
	dev->num_pages
	virtio_notify_config(vdev)  // 给前端发送configuration change interrupt
```



### kernel中的balloon

#### 数据结构

* virtio_balloon_driver

  ```c
  static struct virtio_driver virtio_balloon_driver = {
  	.feature_table = features,
  	.feature_table_size = ARRAY_SIZE(features),
  	.driver.name =	KBUILD_MODNAME,
  	.driver.owner =	THIS_MODULE,
  	.id_table =	id_table,
  	.validate =	virtballoon_validate,
  	.probe =	virtballoon_probe,
  	.remove =	virtballoon_remove,
  	.config_changed = virtballoon_changed,
  #ifdef CONFIG_PM_SLEEP
  	.freeze	=	virtballoon_freeze,
  	.restore =	virtballoon_restore,
  #endif
  };
  ```

  其中feature_table字段定义了balloon的特性

  ​		三个特性： 

  		 - VIRTIO_BALLOON_F_MUST_TELL_HOST
  		 - VIRTIO_BALLOON_F_STATS_VQ
  		 - VIRTIO_BALLOON_F_DEFLATE_ON_OOM

#### 驱动加载

* 几个关键的回调函数注册

  ```c
  // virtio queue的处理函数
  INIT_WORK(&vb->update_balloon_stats_work, update_balloon_stats_func);
  INIT_WORK(&vb->update_balloon_size_work, update_balloon_size_func);
  
  // 几个virtqueue的初始化
  init_vqs(vb)
      -> inflate_vq   <--> callback balloon_ack
      -> deflate_vq   <--> callback balloon_ack
      -> stats_vq     <--> callback stats_request
  
  
  // 热迁移的函数
  vb->vb_dev_info.migratepage = virtballoon_migratepage;
  
  // oom notify 
  vb->nb.notifier_call = virtballoon_oom_notify;   // oom notifier
  vb->nb.priority = VIRTBALLOON_OOM_NOTIFY_PRIORITY;
  err = register_oom_notifier(&vb->nb);
  ```

  tip： 如果没有feature VIRTIO_BALLOON_F_STATS_VQ，就只有两个virt queue



#### virt queue回调函数

* **`inflate_vq`和`deflate_vq`的回调函数**

  ```c
  // balloon_ack
  static void balloon_ack(struct virtqueue *vq)
  {
  	struct virtio_balloon *vb = vq->vdev->priv;
  
  	wake_up(&vb->acked);  // waiting for the host to ack the pages released
  }
  ```

* **`stats_vq`的回调函数**

  > 大部分的回调函数，是guest将请求放入vring中，host处理完请求通知guest
  >
  > stats_vq是host请求，guest将最新的status放入virtqueue中

  ```c
  static void stats_request(struct virtqueue *vq)
  {
  	struct virtio_balloon *vb = vq->vdev->priv;
  
  	spin_lock(&vb->stop_update_lock);
  	if (!vb->stop_update)
  		queue_work(system_freezable_wq, &vb->update_balloon_stats_work);
  	spin_unlock(&vb->stop_update_lock);
  }
  ```

  这里queue_work实际调用的函数是vb->update_balloon_stats_work = **update_balloon_stats_func**.

  ```c
  stats_handle_request
  	-> update_balloon_stats_func
  		-> stats_handle_request
  		
  static void stats_handle_request(struct virtio_balloon *vb)
  {
  	struct virtqueue *vq;
  	struct scatterlist sg;
  	unsigned int len, num_stats;
  
  	num_stats = update_balloon_stats(vb);
  
  	vq = vb->stats_vq;
  	if (!virtqueue_get_buf(vq, &len))
  		return;
  	sg_init_one(&sg, vb->stats, sizeof(vb->stats[0]) * num_stats);
  	virtqueue_add_outbuf(vq, &sg, 1, vb, GFP_KERNEL);
  	virtqueue_kick(vq);  // 通知后端
  }
  ```

  **`stats_handle_request`函数**

  	* `update_balloon_stats`函数：更新virtio-balloon结构体中的stats字段，返回更新的数目（更新3个或者7个）
  	* 构造scatterlist（指向vb->stats）
  	* 放入virtqueue中
  	* 通知host



#### configuration-change回调函数

* config发生改变时调用的函数**`virtballoon_changed`**

  > 在驱动加载时注册
  >
  > `.config_changed = virtballoon_changed`

  实际调用函数`vb->update_balloon_size_work = update_balloon_size_func `

  ```c
  static void update_balloon_size_func(struct work_struct *work)
  {
  	struct virtio_balloon *vb;
  	s64 diff;
  
  	vb = container_of(work, struct virtio_balloon,
  			  update_balloon_size_work);
  	diff = towards_target(vb);  // target - vb->num_pages
  
  	if (diff > 0)
  		diff -= fill_balloon(vb, diff);  // 气球膨胀，并减小diff的值
  	else if (diff < 0)
  		diff += leak_balloon(vb, -diff); // 气球收缩，并增加diff的值
  	update_balloon_size(vb);			 // 更新virtio_balloon_config中的(actual,_)
  										 // 与balloon中的保持一致
  	if (diff)
  		queue_work(system_freezable_wq, work);
  }
  ```

  其中`fill_balloon`函数与`leak_balloon`函数调用`balloon_compaction.c`中的`balloon_page_enqueue()`函数。



### PCI& MMIO

#### qemu后端初始化



```c
// virtio-balloon.c
virtio_balloon_device_realize()
	// 设置device_id, config_vector, vq[0].vector, vq[1].vector, vq[2].vector
	-> virtio_init
	
// handle_output 和 receive_stats函数中
static void balloon_stats_poll_cb(void *opaque)
{
	...
	virtio_notify(vdev, s->svq)
}
static void virtio_balloon_handle_output(VirtIODevice *vdev, VirtQueue *vq)
{
    ...
    virtio_notify(vdev, vq)
}
static void virtio_balloon_receive_stats(VirtIODevice *vdev, VirtQueue *vq)
{
    ...
    virtio_notify(vdev, vq)
}
static void virtio_balloon_to_target(void *opaque, ram_addr_t target)
{
    ...
    virtio_notify_config(vdev);
}
```

```c
virtio_notify
virtio_notify_config
	-> virtio_notify_vector   // config_vector 或者 virtqueue_vector
		// 其中k -> Bus, PciBus 或者 MmioBus
		-> k->notify(qbus->parent, vector); 
```

* PCI Bus

  ```c
  virtio_pci_bus_class_init
  {
  	...
  	k->notify = virtio_pci_notify;
  }
  
  // 其中中断注入的核心函数
  // vector为
  static void virtio_pci_notify(DeviceState *d, uint16_t vector)
  {
      VirtIOPCIProxy *proxy = to_virtio_pci_proxy_fast(d);
  
      if (msix_enabled(&proxy->pci_dev))
          msix_notify(&proxy->pci_dev, vector);   /* msix_notify通过写pci配置空间来传递中断 */
      else {
          VirtIODevice *vdev = virtio_bus_get_device(&proxy->bus);
          pci_set_irq(&proxy->pci_dev, vdev->isr & 1);   /* 通过irq传递中断 */
      }
  }
  ```

* MMIO Bus

  ```c
  static void virtio_mmio_bus_class_init(ObjectClass *klass, void *data)
  {
  	...
      k->notify = virtio_mmio_update_irq;
  }
  
  // 其中中断注入的核心函数
  static void virtio_mmio_update_irq(DeviceState *opaque, uint16_t vector)
  {
      VirtIOMMIOProxy *proxy = VIRTIO_MMIO(opaque);
      VirtIODevice *vdev = virtio_bus_get_device(&proxy->bus);
      int level;
  
      if (!vdev) {
          return;
      }
      level = (vdev->isr != 0);
      DPRINTF("virtio_mmio setting IRQ %d\n", level);
      qemu_set_irq(proxy->irq, level);   // 调用irq->handler
  }
  ```

  

#### Guest前端初始化

```c
virtballoon_probe
	// 初始化virtio-balloon对应的三个virtqueue
	-> init_vqs
		-> vb->vdev->config->find_vqs(vb->vdev, nvqs, vqs, callbacks, names);
```

下面看PCI总线协议和MMIO总线协议下find_vqs的不同

* PCI总线协议

  ```c
  static const struct virtio_config_ops virtio_pci_config_nodev_ops = {
  	...
  	.find_vqs	= vp_modern_find_vqs,
  }
  
  // 核心函数
  vp_modern_find_vqs
  	-> vp_find_vqs
  		-> vp_try_to_find_vqs
  ```

  看一下具体实现的函数：

  * 优先为config 和 每个queues 单独分配msix
  * 分配不成功，则为config分配一个msix, 为所有queues分配一个msix(共享)
  * least choice: 分配irq(共享) 

  ```c
  /* the config->find_vqs() implementation */
  int vp_find_vqs(struct virtio_device *vdev, unsigned nvqs,
  		struct virtqueue *vqs[],
  		vq_callback_t *callbacks[],
  		const char * const names[])
  {
  	int err;
  
  	/* Try MSI-X with one vector per queue. */
  	err = vp_try_to_find_vqs(vdev, nvqs, vqs, callbacks, names, true, true);
  	if (!err)
  		return 0;
  	/* Fallback: MSI-X with one vector for config, one shared for queues. */
  	err = vp_try_to_find_vqs(vdev, nvqs, vqs, callbacks, names,
  				 true, false);
  	if (!err)
  		return 0;
  	/* Finally fall back to regular interrupts. */
  	return vp_try_to_find_vqs(vdev, nvqs, vqs, callbacks, names,
  				  false, false);
  }
  
  /*
  static int vp_try_to_find_vqs(struct virtio_device *vdev, unsigned nvqs,
  			      struct virtqueue *vqs[],
  			      vq_callback_t *callbacks[],
  			      const char * const names[],
  			      bool use_msix,   // 使用msix 或者 irq
  			      bool per_vq_vectors)  // 是否共享
  */
  ```

  

* MMIO总线协议

  ```c
  static const struct virtio_config_ops virtio_mmio_config_ops = {
  	...
  	.find_vqs	= vm_find_vqs,
  };
  
  // 核心函数 vm_find_vqs
  find_vqs
  	-> vm_find_vqs
  		-> request_irq(irq, vm_interrupt, IRQF_SHARED, dev_name(&vdev->dev), vm_dev);
  ```

  * 这里irq的申请函数`request_irq`，注意参数`IRQF_SHARED`,即该设备的config和所有virt-queue使用同一个irq.

  * 中断处理函数为`vm_interrupt`，看一下函数的具体实现：

    会通过`status`判断是configuration change interrupt / vring interrupt

    ```c
    /* Notify all virtqueues on an interrupt. */
    static irqreturn_t vm_interrupt(int irq, void *opaque)
    {
    	struct virtio_mmio_device *vm_dev = opaque;
    	struct virtio_mmio_vq_info *info;
    	unsigned long status;
    	unsigned long flags;
    	irqreturn_t ret = IRQ_NONE;
    
    	/* Read and acknowledge interrupts */
    	status = readl(vm_dev->base + VIRTIO_MMIO_INTERRUPT_STATUS);
    	writel(status, vm_dev->base + VIRTIO_MMIO_INTERRUPT_ACK);
    
    	if (unlikely(status & VIRTIO_MMIO_INT_CONFIG)) {
    		virtio_config_changed(&vm_dev->vdev);     // config change interrupt
    		ret = IRQ_HANDLED;
    	}
    
    	if (likely(status & VIRTIO_MMIO_INT_VRING)) {   // vring interrupt
    		spin_lock_irqsave(&vm_dev->lock, flags);
    		list_for_each_entry(info, &vm_dev->virtqueues, node)
    			ret |= vring_interrupt(irq, info->vq);   
    		spin_unlock_irqrestore(&vm_dev->lock, flags);
    	}
    
    	return ret;
    }
    ```

    