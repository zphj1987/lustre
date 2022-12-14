# mds的备份恢复-ldiskfs方案

## 背景
mds与mgs的备份方法基本类似的，从使用过程来说，mgs的数据变动没有mds的变动那么频繁，所以两者从备份密度来说，如果mds的数据备份的尽量要多备份会比较好

## 备份思路
有的备份方案需要离线备份数据，这里利用了lvm能够做快照的属性，来进行备份
- 1、对数据进行快照并挂载起来
- 2、提取快照数据
- 3、恢复快照数据到新的存储
- 4、恢复业务
日常备份就是操作步骤1和步骤2即可

由于备份需要在线做备份，所以需要利用lvm的快照功能，创建一个快照，把快照的存储挂载起来，进行数据的备份提取

## 具体操作方法
### 创建lvm用于mds
```bash
[root@test201 ~]#  pvcreate /dev/sdc1
  Physical volume "/dev/sdc1" successfully created.
[root@test201 ~]# vgcreate mdsvg /dev/sdc1
  Volume group "mdsvg" successfully created
[root@test201 ~]# lvcreate -L 10G -n mdslv mdsvg
  Logical volume "mdslv" created.
```
### 创建mds
```bash
[root@test201 ~]# mkfs.lustre --fsname=lustrefs --mgsnode=192.168.0.201@tcp0  --mdt --index=0 /dev/mdsvg/mdslv
[root@test201 ~]# mount -t lustre /dev/mdsvg/mdslv /lustre/mds0/
```
写入一定量的数据，然后再进行快照相关的操作

### 创建快照
```bash
[root@test201 ~]# lvcreate -L 5G --snapshot --name mdslv_snap001 /dev/mdsvg/mdslv
  Logical volume "mdslv_snap001" created.
```
### 提取快照数据
```bash
[root@test201 ~]# mount -t ldiskfs  /dev/mdsvg/mdslv_snap001 /opt/mdsbak/
[root@test201 ~]# cd /opt/mdsbak/
[root@test201 mds]# tar czvf ../mds.back.tgz --xattrs --xattrs-include="trusted.*" --sparse .
```
再写入一定量的数据，模拟快照与实际的使用存在时间差，我们做恢复，也是只能恢复到快照的时间点的数据

### 新创建mds
创建lvm
```bash
[root@test201 opt]# lvcreate -L 2G -n newmdslv mdsvg
  Logical volume "newmdslv" created.
```

格式化设备
```bash
[root@test201 opt]# mkfs.lustre --fsname=lustrefs --mgsnode=192.168.0.201@tcp0  --mdt --index=0  --replace  /dev/mdsvg/newmdslv
[root@test201 opt]# mount -t ldiskfs /dev/mdsvg/newmdslv /opt/mds
```

### 恢复数据
```bash
[root@test201 opt]# cd /opt/mds
[root@test201 mds]# tar xzvpf ../mds.back.tgz --xattrs --xattrs-include="trusted.*" --sparse
[root@test201 mds]# rm -rf oi.16* lfsck_* LFSCK
[root@test201 mds]# rm -f CATALOGS
[root@test201 opt]# umount /opt/mds
```
上面的几个删除的数据是官方文档推荐删除的，说是避免恢复中断（参考18.4）

### 挂载mds
```bash
[root@test201 opt]# mount -t lustre /dev/mapper/mdsvg-newmdslv /lustre/mds0/
```

### 验证是否恢复
验证就直接去目录进行list操作即可，如果有异常的情况是无法列出的