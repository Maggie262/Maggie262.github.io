---
layout:     post
title:      "hugepage configuration"
subtitle:   "  "
date:       2020-02-24
author:     "Maggie"
header-img: "img/post-bg-digital-native.jpg"
catalog: true
tags:
    - linux
    - memory
---



#### hugepages

* 查看大页内存配置

  ```shell
  cat /sys/kernel/mm/transparent_hugepage/enabled
  ```

  * `always`
  * `madvise`: 避免改变内存占用
  * `never`

  ```shell
  cat /proc/meminfo
  ```

  

* 启用大页

  ```shell
  # 设置大页数目
  sysctl vm.nr_hugepages=1024
  # 挂载大页, hugetlbfs是一个虚拟文件系统，它运行在内核层，不与磁盘空间相对应。
  mount -t hugetlbfs hugetlbfs /dev/hugepages
  ```

* 禁用大页

  ```shell
  sysctl vm.nr_hugepages=0
  unmount hugetlbfs
  ```

  



#### 虚拟机采用大页

**qemu命令**

```shell
qemu-system-x86_64 -enable-kvm -m 4096 -smp 4  -spp on -monitor pty -cpu host \
-device virtio-net-pci,netdev=nic0,mac=00:16:3e:0c:12:78 \
-netdev tap,id=nic0,script=/etc/kvm/qemu-ifup \
-drive file=/share/xvs/var/rhel7.qcow,if=none,id=virtio-disk0 \
-device virtio-blk-pci,drive=virtio-disk0 \
-mem-path /dev/hugepages

qemu-system-x86_64 -m 1024 -smp 2 rhel6u3.img -mem-path /dev/hugepages -mem-prealloc 
# -mem-prealloc 预先分配好
```

在host上执执行`cat /proc/meminfo`查看大页的使用情况，发现已使用的大页数目乘2M，与虚拟机规格不相等， 这是因为启动虚拟机时并没有实际分配全部的内存，如果设置参数`-mem-prealloc`，则两个数值一致



* 虚拟机使用大页，可以提高性能。但是大页不能换出， balloon设备不能使用





#### KSM内存页合并技术

KSM：主要将相同内存页进行合并，CentOS 6和Centos 7默认打开，主要有两个服务
 KSM服务，ksmtuned服务

开启服务并开启开机自启

```bash
systemctl start ksm
systemctl start ksmtuned
systemctl enable kvm
systemctl enable ksmtuned
```

检测：查看/sys/kernel/mm/ksm/目录下文件

```undefined
pages_shared：正在共享的内存页
pages_sharing：多少节点被共享并且被保存多少
pages_unshared：内存被合并时有多少内存页独特但是反复被检查
pages_volatile:：多少内存页改变太快被放置
full_scans:：对少次可以合并区域被扫描
```

**阻止个别虚拟机进行内存压缩的方法**
 使用nosharepages关键字阻止宿主机将特定的虚拟机内存页合并，xml配置文件如下：

```xml
<memoryBacking>
        <nosharepages/>
</momoryBacking>
```

应用场景：
 测试环境推荐使用，生产环境慎用
 开启kvm技术会导致两个结果：
 一是会消耗一定的计算机资源用于内存扫描，加重CPU的消耗
 二是内存超用，当内存不够的时候，只能频繁地使用swap交互，导致虚拟机性能下降





[**qemu-system-x86_64命令总结**]([http://blog.leanote.com/post/7wlnk13/%E5%88%9B%E5%BB%BAKVM%E8%99%9A%E6%8B%9F%E6%9C%BA](http://blog.leanote.com/post/7wlnk13/创建KVM虚拟机))