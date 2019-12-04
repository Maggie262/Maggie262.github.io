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
    - UX/UI
---

### virtio balloon



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





