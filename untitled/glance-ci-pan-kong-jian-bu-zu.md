# glance磁盘空间不足

#### 问题

glance磁盘空间空间不足（分区占用100%），导致服务异常。

#### 原因

初始情况下，镜像数据很少，glance服务所在机器磁盘够用。随着数据增长，磁盘数据量增大。

#### 解决方案

**增加glance分区大小**

glance存储层采用swift对象存储，部署时专为glance服务划分了逻辑分区`/dev/mapper/image-glance`,挂载在`/var/lib/glance`目录，因此使用LVM扩展逻辑分区即可。  
假设机器上新增了磁盘设备`/dev/sdb`。

1. 对磁盘进行分区

```text
fdisk /dev/sdb
n
p
1
\n
\n
t
8e
w
```

```text
 mkfs.xfs /dev/sdb1 
```

1. 扩展卷

```text
pvcreate /dev/sdb1
vgextend image /dev/sdb1
lvextend -L 1.1t /dev/mapper/image-glance 
xfs_growfs /dev/mapper/image-glance
```

