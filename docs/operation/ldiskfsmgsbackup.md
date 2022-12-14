# mgs的备份恢复-ldiskfs方案

## 背景

lustre内部主要有三个角色，mgs,mds,osd,这三个数据里面，mgs的数据占用最小，mds多一点，osd最多
对于整套环境来说，虽然底层能够通过做raid方式来进行数据的安全性的加固，但是如果能够多备份一些数据，对于某些极端场景还是很有用的
osd的数据因为存储的是真实的数据块，所以备份起来需要的数据量太大，所以对数据量不大的数据，我们最好能够定期备份，所以提供mgs和mds的两种数据的备份方法

## 备份思路
有的备份方案需要离线备份数据，这里利用了lvm能够做快照的属性，来进行备份
- 1、对数据进行快照并挂载起来
- 2、提取快照数据
- 3、恢复快照数据到新的存储
- 4、恢复业务
日常备份就是操作步骤1和步骤2即可

由于备份需要在线做备份，所以需要利用lvm的快照功能，创建一个快照，把快照的存储挂载起来，进行数据的备份提取

## 具体操作方法
因为要利用到快照的功能，所以在做存储的时候，需要提前做好lvm，并且因为快照是需要从vg里面分配一定的空间的，所以在做lvm的时候，需要提前预留一定的空间给创建快照使用，因为mgs的数据不怎么变化，所以快照的预留大小可以小点也是没有问题的，lvm的快照遵循的原则是，变化的数据大小不能超过创建的快照的卷的大小，否则快照就损坏了

### 创建lvm用于mgs
```bash
[root@test201 ~]# pvcreate /dev/sdb1
  Physical volume "/dev/sdb1" successfully created.
[root@test201 ~]# vgcreate mgsvg /dev/sdb1
  Volume group "mgsvg" successfully created
[root@test201 ~]# lvcreate -L 10G -n mgslv mgsvg
  Logical volume "mgslv" created.
```
上面就创建好了lvm的设备

### 创建mgs
```bash
[root@test201 ~]# mkfs.lustre --fsname=lustrefs --mgs /dev/mgsvg/mgslv

   Permanent disk data:
Target:     MGS
Index:      unassigned
Lustre FS:  lustrefs
Mount type: ldiskfs
Flags:      0x64
              (MGS first_time update )
Persistent mount opts: user_xattr,errors=remount-ro
Parameters:

checking for existing Lustre data: not found
device size = 10240MB
formatting backing filesystem ldiskfs on /dev/mgsvg/mgslv
  target name   MGS
  kilobytes     10485760
  options        -q -O uninit_bg,dir_nlink,quota,huge_file,large_dir,flex_bg -E lazy_journal_init -F
mkfs_cmd = mke2fs -j -b 4096 -L MGS  -q -O uninit_bg,dir_nlink,quota,huge_file,large_dir,flex_bg -E lazy_journal_init -F /dev/mgsvg/mgslv 10485760k
Writing CONFIGS/mountdata
```
### 挂载mgs
```bash
[root@test201 ~]# mkdir -p /lustre/mgs
[root@test201 ~]# mount -t lustre  /dev/mgsvg/mgslv  /lustre/mgs
```

### 创建快照
```bash
[root@test201 ~]# lvcreate -L 5G --snapshot --name mgslv_snap001 /dev/mgsvg/mgslv
  Logical volume "mgslv_snap001" created.
```
这个快照大小可以自己指定，只要不超过提取过程中的数据变化大小就行
### 挂载快照
```bash
[root@test201 ~]# mount -t ldiskfs /dev/mgsvg/mgslv_snap001 /opt/mgsbk/
[root@test201 ~]# cd /opt/mgsbk/
[root@test201 ~]# tar czvf ../mgs.back.tgz --xattrs --xattrs-include="trusted.*" --sparse .
```
注意上面的扩展属性一定要带上，这个记录了一些值，需要用到的，我们用的centos7的tar命令是支持存储扩展属性的

### 创建一个新的lvm
```bash
[root@test201 ~]# lvcreate -L 1G -n newmgslv mgsvg
  Logical volume "newmgslv" created.
```
这个用新设备创建就行，这里是模拟的数据损坏，就直接用之前的vg了，并且因为mgs大小不大，就给个小空间即可

### 格式化新设备
```bash
[root@test201 ~]# mkfs.lustre --fsname=lustrefs --mgs  --replace  /dev/mgsvg/newmgslv
```
### 挂载新设备
```bash
[root@test201 ~]# mount -t ldiskfs /dev/mgsvg/newmgslv /opt/mgs/
```
### 还原备份的数据
```bash
[root@test201 ~]# cd  /opt/mgs/
[root@test201 ~]# tar -xzvpf ../mgs.back.tgz --xattrs --xattrs-include="trusted.*" --sparse
[root@test201 ~]# umount /opt/mgs/
```
到这里相关的数据就还原回来了，直接再进行挂载即可

### 挂载mgs
```bash
[root@test201 ~]# mount -t lustre /dev/mgsvg/newmgslv /lustre/mgs
```

## 总结
实际使用过程中，我们主要是要关注下备份的相关的操作
- 创建快照并提取数据
- 删除快照

上面三个数据的大小非常小，不占什么空间，但是非常重要，所以我们建议生产是定期备份的，快照本身只是记录了在某个时间点的数据情况，数据设备本身如果损坏，快照也是损坏的，所以我们需要把数据提取出来，快照只是一个临时数据的作用，所以lvm的快照在提取完了以后是可以删除的


