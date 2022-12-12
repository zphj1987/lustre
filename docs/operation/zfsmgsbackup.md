# mgs的备份恢复-zfs方案

## 背景

lustre内部主要有三个角色，mgs,mds,osd,这三个数据里面，mgs的数据占用最小，mds多一点，osd最多
对于整套环境来说，虽然底层能够通过做raid方式来进行数据的安全性的加固，但是如果能够多备份一些数据，对于某些极端场景还是很有用的
osd的数据因为存储的是真实的数据块，所以备份起来需要的数据量太大，所以对数据量不大的数据，我们最好能够定期备份，所以提供mgs和mds的两种数据的备份方法

## 备份思路
有的备份方案需要离线备份数据，这里利用了zfs能够做快照的属性，来进行备份
- 1、对数据进行快照
- 2、导出快照和元数据
- 3、恢复快照和元数据到新的存储
- 4、恢复业务
日常备份就是操作步骤1和步骤2即可


## 具体操作方法

### 备份zfs的元数据
lustre是利用zfs的元数据进行的挂载和系统判断的，这个跟ceph里面利用lvm记录节点的信息类似的处理

```bash
zpool get all  mgspool > mgspool.attr
zfs get -rHp all mgspool > mgspool.mgt.attr
```

### 备份zfs的数据
```bash
[root@test201 ~]# zfs snapshot mgspool/mgt@1
[root@test201 ~]# zfs send mgspool/mgt@1 > zfs.mgspool.bk
```

我们假设现在mgs对应的存储池完全损坏了，我们就当没有这个数据了，然后继续后面的恢复流程

### 恢复MGS相关的数据

#### 创建一个新的zpool存储池

```bash
[root@test201 ~]# zpool create -O canmount=off -o multihost=on -o cachefile=none mgsnewpool sdd1
```

#### 恢复备份的mgs的内部数据
```bash
[root@test201 ~]# zfs receive mgsnewpool/mgt < zfs.mgspool.bk
```
这个操作以后数据是恢复了，但是元数据没有恢复，lustre是根据zfs的属性来进行判断挂载的，也就是下面这些数据
```bash
[root@test201 ~]# cat mgspool.mgt.attr |grep local
mgspool	canmount	off	local
mgspool/mgt	canmount	off	local
mgspool/mgt	xattr	sa	local
mgspool/mgt	dnodesize	auto	local
mgspool/mgt	lustre:flags	36	local
mgspool/mgt	lustre:fsname	lustrefs	local
mgspool/mgt	lustre:version	1	local
mgspool/mgt	lustre:svname	MGS	local
mgspool/mgt	lustre:index	65535	local
```
带local的就是修改过默认值的，也就是lustre这边设置过属性值的，我们根据这些备份信息进行恢复
```bash
[root@test201 ~]# zfs set canmount=off mgsnewpool/mgt
[root@test201 ~]# zfs set xattr=sa mgsnewpool/mgt
[root@test201 ~]# zfs set dnodesize=auto mgsnewpool/mgt
[root@test201 ~]# zfs set lustre:flags=36 mgsnewpool/mgt
[root@test201 ~]# zfs set lustre:fsname=lustrefs mgsnewpool/mgt
[root@test201 ~]# zfs set lustre:version=1 mgsnewpool/mgt
[root@test201 ~]# zfs set lustre:svname=MGS mgsnewpool/mgt
[root@test201 ~]# zfs set lustre:index=65535 mgsnewpool/mgt
```
现在数据和元数据都进行了恢复了，那么这个新的存储池和mgs的目录就能够挂载了

### 挂载新的mgs
```bash
[root@test201 ~]# mount -t lustre  mgsnewpool/mgt /lustre/mgs
```
要验证新的mgs是否生效，就把客户端卸载掉后重新挂载一下,如果mgs是异常的，那么客户端是挂载不上的

## 总结
实际使用过程中，我们主要是要关注下备份的相关的操作
- 备份元数据
- 备份数据
- 导出数据

上面三个数据的大小非常小，不占什么空间，但是非常重要，所以我们建议生产是定期备份的

