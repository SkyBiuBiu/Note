# [Linux] 红帽高级存储功能 - Stratis与VDO

[toc]

## Stratis 卷管理文件系统

### 介绍



![image-20220413143920555](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220413143920555.png)

红帽的 Stratis 是新一代的存储管理解决方案，称为卷管理文件系统。可以通过它创建文件系统及调整其大小时以动态、透明的方式来管理卷层。

Stratis 以管理物理存储池的服务形式运行，并透明地为所创建的文件系统创建和管理卷。由于 Stratis 使用现有的存储驱动程序和工具，因此 Stratis 也支持当前在 lvm、xfs 和设备映射器中使用的所有高级存储功能。Stratis 文件系统没有固定大小，也不再预分配未使用的块空间。

Stratis 使用存储的元数据来识别所管理的池、卷和文件系统。因此绝不应该对 Stratis 创建的文件系统进行手动重新格式化或重新配置；只应使用 Stratis 工具和命令对它们进行管理。手动配置 Stratis 文件系统可能会导致该元数据丢失，并阻止 Stratis 识别它已创建的文件系统。您可以使用不同组的块设备来创建多个池。在每个池中，您可以创建一个或多个文件系统。目前每个池最多可以创建 2^24 个文件系统。

![img](https://img-blog.csdnimg.cn/2021042107221059.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dhb2ZlaTA0Mjg=,size_16,color_FFFFFF,t_70)

在内部 Stratis 使用 Backstore 子系统来管理块设备，并使用 Thinpool 子系统来管理池。Backstore 有一个数据层，负责维护块设备磁盘上的元数据，以及检测和纠正数据损坏。缓存层使用高性能块设备作为数据层之上的缓存。Thinpool 子系统管理与 Stratis 文件系统关联的精简部署卷。该子系统使用 dm-tin设备映射器驱动程序取代 LVM 进行虚拟卷大小调整和管理。dm-thin 可以创建虚拟大小比较大，但物理大小比较小的卷, 采用 XFS 格式。当物理大小快要满时，Stratis 会自动将其扩大。



### 特点

使用Stratis管理分层存储，有以下的好处：

- 自动精简配置（超额分配）
- 文件系统快照
- 基于池的存储管理
- 存储监控

Stratis相关概念：

- blockdev：块设备
- pool：存储池
- filesystem：文件系统

对应关系：

- 1个Pool，最多可以创建2²⁴个文件系统，自动分配大小，当总数据量接近虚拟分配的大小则自动扩容，只要存储池物理设备充足，理论上无限扩展。



### 使用

1. 安装软件包

```bash
[root@localhost /]# yum install -y stratis-cli stratisd
```

2. 启动服务并设置服务开机自启

```bash
[root@localhost /]# systemctl enable --now stratisd
```

3. 添加一块SATA硬盘，创建存储池

```bash
[root@localhost ~]# stratis pool create pool1 /dev/sdb
```

4. 查看可用池列表

```bash
[root@localhost ~]# stratis pool list
Name                  Total Physical   Properties                                   UUID
pool1   5 GiB / 37.63 MiB / 4.96 GiB      ~Ca,~Cr   d45d2be9-e1ca-4700-a03f-365eda2eac34
```

5. 再次添加一块硬盘，添加这个块设备到存储池

```bash
[root@localhost ~]# stratis pool add-data pool1 /dev/sdc
```

6. 查看池的块设备

```bash
[root@localhost ~]# stratis blockdev list
Pool Name   Device Node   Physical Size   Tier
pool1       /dev/sdb              5 GiB   Data
pool1       /dev/sdc              5 GiB   Data

[root@localhost ~]# stratis pool list
Name                   Total Physical   Properties                                   UUID
pool1   10 GiB / 41.63 MiB / 9.96 GiB      ~Ca,~Cr   d45d2be9-e1ca-4700-a03f-365eda2eac34
```

7. 为池创建动态、灵活的文件系统

```bash
[root@localhost ~]# stratis filesystem create pool1 fs1

```

8. 查看文件系统

```bash
[root@localhost ~]# stratis filesystem list
Pool Name   Name   Used      Created             Device                   UUID            
pool1       fs1    546 MiB   Apr 13 2022 15:22   /dev/stratis/pool1/fs1   b28ec1f4-21ed-4d89-84f5-d8ca91ef12ce

[root@localhost ~]# blkid /dev/stratis/pool1/fs1
/dev/stratis/pool1/fs1: UUID="b28ec1f4-21ed-4d89-84f5-d8ca91ef12ce" TYPE="xfs"
```

9. 挂载文件系统

这里的 x-systemd.requires=stratisd.service，是通过 systemd 来确保 stratisd.service（Stratis服务的守护进程）启动，再挂载这个文件系统，否则电脑将无法开机。 

```bash
[root@localhost ~]# vim /etc/fstab
...
UUID=b28ec1f4-21ed-4d89-84f5-d8ca91ef12ce /dir1 xfs     defaults,x-systemd.requires=stratisd.service    0 0

[root@localhost ~]# mount -a

[root@localhost ~]# df -hT
Filesystem                                                                                      Type      Size  Used Avail Use% Mounted on
...
/dev/mapper/stratis-1-d45d2be9e1ca4700a03f365eda2eac34-thin-fs-b28ec1f421ed4d8984f5d8ca91ef12ce xfs       1.0T  7.2G 1017G   1% /dir1
```

10. 创建文件测试

```bash
[root@localhost ~]# echo hello world > /dir1/file1
```

11. 创建快照

```bash
[root@localhost ~]# stratis filesystem snapshot pool1 fs1 snap1

[root@localhost ~]# stratis filesystem list
Pool Name   Name    Used      Created             Device                     UUID         
pool1       fs1     546 MiB   Apr 13 2022 15:22   /dev/stratis/pool1/fs1     b28ec1f4-21ed-4d89-84f5-d8ca91ef12ce
pool1       snap1   546 MiB   Apr 13 2022 15:31   /dev/stratis/pool1/snap1   6dafb135-a4f8-49a8-a757-0c78cb430fb2

[root@localhost ~]# rm -rf /dir1/file1 

[root@localhost ~]# mount /dev/stratis/pool1/snap1 /dir2

[root@localhost ~]# cat /dir2/file1
```



## VDO 虚拟数据优化器

### 介绍

LVM只能解决容量的问题，但不具备数据压缩的能力。

VDO（Virtual Data Optimize）：虚拟数据优化器，通过压缩或删除存储设备上的数据来优化空间，VDO层放置在现有块存储设备上层，例如RAID设备或本地磁盘的顶部。



### 特点

- kvdo：用于控制数据压缩
- uds：用于重复数据删除

vdo不仅能对数据进行压缩，还能对重复的数据进行优化。

Linux下依赖vdo.service服务，否则是不能使用vdo的



### 使用 

1. 安装软件包（默认已安装）

```bash
[root@localhost ~]# yum install -y vdo
```

2. 创建VDO卷

```bash
[root@localhost ~]# vdo create --name=vdo1 --device=/dev/sdb --vdoLogicalSize=10G
Creating VDO vdo1
      The VDO volume can address 6 GB in 3 data slabs, each 2 GB.
      It can grow to address at most 16 TB of physical storage in 8192 slabs.
      If a larger maximum size might be needed, use bigger slabs.
Starting VDO vdo1
Starting compression on VDO vdo1
VDO instance 0 volume is ready at /dev/mapper/vdo1
```

3. 查看VDO卷的属性与状态

```bash
[root@localhost ~]# vdo status --name=vdo1
VDO status:
  Date: '2022-04-13 16:34:12+08:00'
  Node: localhost.localdomain
Kernel module:
  Loaded: true
  Name: kvdo
  Version information:
    kvdo version: 6.2.2.117
Configuration:
  File: /etc/vdoconf.yml
  Last modified: '2022-04-13 16:33:40'
VDOs:
  vdo1:
    Acknowledgement threads: 1
    Activate: enabled
    Bio rotation interval: 64
    Bio submission threads: 4
    Block map cache size: 128M
    Block map period: 16380
    Block size: 4096
    CPU-work threads: 2
    Compression: enabled
    Configured write policy: auto
    Deduplication: enabled
    Device mapper status: 0 20971520 vdo /dev/sdb normal - online online 1049638 2621440
    Emulate 512 byte: disabled
    Hash zone threads: 1
    Index checkpoint frequency: 0
    Index memory setting: 0.25
    Index parallel factor: 0
    Index sparse: disabled
    Index status: online
    Logical size: 10G
    Logical threads: 1
    Max discard size: 4K
    Physical size: 10G
    Physical threads: 1
    Slab size: 2G
    Storage device: /dev/sdb
    UUID: VDO-59a6cb34-0ba6-4e3d-81b8-967710ca668f
    VDO statistics: not available
```

4. 显示VDO卷列表

```bash
[root@localhost ~]# vdo list
vdo1
```

5. 停止和启动VDO卷

```bash
[root@localhost ~]# vdo stop -n vdo1
Stopping VDO vdo1

[root@localhost ~]# vdo start -n vdo1
Starting VDO vdo1
Starting compression on VDO vdo1
VDO instance 1 volume is ready at /dev/mapper/vdo1

```

6. 查看VDO卷上是否启用了压缩（Compression）和重复数据（Deduplication）删除功能

```bash
[root@localhost ~]# vdo status --name=vdo1 | grep Dedu
    Deduplication: enabled


[root@localhost ~]# vdo status --name=vdo1 | grep Com
    Compression: enabled
```

7. 格式化VDO卷

```bash
[root@localhost ~]# mkfs.xfs -K /dev/mapper/vdo1
meta-data=/dev/mapper/vdo1       isize=512    agcount=4, agsize=655360 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=2621440, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

8. 检查设备事件处理是否完成

```bash
[root@localhost ~]# udevadm settle
```

9. 创建挂载点，设置开机自动挂载

```bash
[root@localhost ~]# mkdir /mnt/vdo1

[root@localhost ~]# blkid | grep vdo1
/dev/mapper/vdo1: UUID="b4373f24-2c29-4893-bce3-fb608025107d" TYPE="xfs"

[root@localhost ~]# vim /etc/fstab
...
UUID=b4373f24-2c29-4893-bce3-fb608025107d /mnt/vdo1 xfs defaults,x-systemd.requires=vdo.service 0 0

[root@localhost ~]# mount /mnt/vdo1/
```

10. 查看卷初始信息

```bash
[root@localhost /]# vdostats --human-readable
Device                    Size      Used Available Use% Space saving%
/dev/mapper/vdo1         10.0G      4.0G      6.0G  40%           N/A
```

11. 准备一个大文件用于测试

```bash
[root@localhost /]# dd if=/dev/urandom of=/root/testfile1 bs=1M count=300
300+0 records in
300+0 records out
314572800 bytes (315 MB, 300 MiB) copied, 12.8609 s, 24.5 MB/s

[root@localhost /]# ls -lh /root/testfile1 
-rw-r--r--. 1 root root 300M Apr 13 17:06 /root/testfile1
```

12. 把文件复制到VDO卷挂载目录

```bash
[root@localhost /]# cp /root/testfile1 /mnt/vdo1/testfile1.1
[root@localhost /]# vdostats --human-readable
Device                    Size      Used Available Use% Space saving%
/dev/mapper/vdo1         10.0G      4.3G      5.7G  42%            3%
```

13. 重复多次操作观察Used与Saving%的变化

```bash
[root@localhost /]# cp /root/testfile1 /mnt/vdo1/testfile1.2
[root@localhost /]# vdostats --human-readable
Device                    Size      Used Available Use% Space saving%
/dev/mapper/vdo1         10.0G      4.3G      5.7G  43%           48%

[root@localhost /]# cp /root/testfile1 /mnt/vdo1/testfile1.3
[root@localhost /]# vdostats --human-readable
Device                    Size      Used Available Use% Space saving%
/dev/mapper/vdo1         10.0G      4.3G      5.7G  43%           65%
```

