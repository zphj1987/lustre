# mds的备份恢复-zfs方案

## 背景
mds与mgs的备份方法基本类似的，从使用过程来说，mgs的数据变动没有mds的变动那么频繁，所以两者从备份密度来说，如果mds的数据备份的尽量要多备份会比较好

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
zpool get all  mdspool > mdspool.attr
zfs get -rHp all mdspool > mdspool.mds0.attr
```

### 备份zfs的数据
```bash
[root@test202 ~]# zfs snapshot mdspool/mds0@1
[root@test202 ~]# zfs send mdspool/mds0@1 > zfs.mdspool.mds0.bk
```


#### 创建一个新的zpool存储池

```bash
[root@test201 ~]# zpool create -O canmount=off -o multihost=on -o cachefile=none mdsnewpool sdc1
```

#### 恢复备份的mds的内部数据
```bash
[root@test202 ~]# zfs receive mdsnewpool/mds0 <  zfs.mdspool.mds0.bk
```
这个操作以后数据是恢复了，但是元数据没有恢复，lustre是根据zfs的属性来进行判断挂载的，也就是下面这些数据

```bash
[root@test202 ~]# cat mdspool.mds0.attr |grep local
mdspool	canmount	off	local
mdspool/mds0	canmount	off	local
mdspool/mds0	xattr	sa	local
mdspool/mds0	dnodesize	auto	local
mdspool/mds0	lustre:flags	33	local
mdspool/mds0	lustre:svname	lustrefs-MDT0000	local
mdspool/mds0	lustre:mgsnode	192.168.0.201@tcp	local
mdspool/mds0	lustre:fsname	lustrefs	local
mdspool/mds0	lustre:version	1	local
mdspool/mds0	lustre:index	0	local
```
带local的就是修改过默认值的，也就是lustre这边设置过属性值的，我们根据这些备份信息进行恢复
```bash
[root@test202 ~]# zfs set canmount=off   mdsnewpool/mds0
[root@test202 ~]# zfs set xattr=sa   mdsnewpool/mds0
[root@test202 ~]# zfs set dnodesize=auto   mdsnewpool/mds0
[root@test202 ~]# zfs set lustre:flags=33   mdsnewpool/mds0
[root@test202 ~]# zfs set lustre:svname=lustrefs-MDT0000   mdsnewpool/mds0
[root@test202 ~]# zfs set lustre:mgsnode=192.168.0.201@tcp   mdsnewpool/mds0
[root@test202 ~]# zfs set lustre:fsname=lustrefs   mdsnewpool/mds0
[root@test202 ~]# zfs set lustre:version=1   mdsnewpool/mds0
[root@test202 ~]# zfs set lustre:index=0   mdsnewpool/mds0
```
### 挂载新的mds
```bash
[root@test202 ~]# mount -t lustre mdsnewpool/mds0 /lustre/mds0/
```

### 验证是否恢复
验证就直接去目录进行list操作即可，如果有异常的情况是无法列出的