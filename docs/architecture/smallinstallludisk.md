# lustre的最小化部署-ludiskfs

## 最小化部署

之前介绍了zfs模式的最小化的部署，本篇是测试ludiskfs的模式的最小化部署

- 1、部署mgs
- 2、部署mds
- 3、部署osd
- 4、部署client
- 5、写数据

## 部署系统

### 准备硬件环境
安装一台centos7的操作系统，更新好内核，只用进行相关的配置
准备硬盘或者分区：

- 1、一个盘给mgs使用
- 2、一个盘给mds使用
- 3、一个盘给osd使用

### 配置网络
```bash
[root@lab105 ~]# cat /etc/modprobe.d/lustre.conf 
options lnet networks=tcp0(eth0)
```
所有节点都需要配置网络，这个是告诉节点是通过哪个网卡进行通信的

### 重启服务
```bash
[root@lab105 ~]# depmod  -a
[root@lab105 ~]# systemctl restart lustre
```
### 加载zfs模块
```bash
modprobe zfs
genhostid 
```

### 配置MGS

#### 创建MGS
```bash
[root@test201 ~]# mkfs.lustre --fsname=lustrefs --mgs  --reformat /dev/sdb1

   Permanent disk data:
Target:     MGS
Index:      unassigned
Lustre FS:  lustrefs
Mount type: ldiskfs
Flags:      0x64
              (MGS first_time update )
Persistent mount opts: user_xattr,errors=remount-ro
Parameters:

device size = 20478MB
formatting backing filesystem ldiskfs on /dev/sdb1
	target name   MGS
	kilobytes     20969472
	options        -q -O uninit_bg,dir_nlink,quota,huge_file,large_dir,flex_bg -E lazy_journal_init -F
mkfs_cmd = mke2fs -j -b 4096 -L MGS  -q -O uninit_bg,dir_nlink,quota,huge_file,large_dir,flex_bg -E lazy_journal_init -F /dev/sdb1 20969472k
Writing CONFIGS/mountdata
```

这个地方指定了lustrefs，后面创建其它的也要指定，这里是单机的，就没有指定备用节点的一些信息

### 挂载mgs

准备一个挂载目录
```bash
[root@lab105 ~]# mkdir /lustre/mgs -p
```
挂载
```bash
[root@test201 ~]# mount -t lustre /dev/sdb1 /lustre/mgs
```
mgs的服务就创建完毕了,后面创建mds和osd的流程和命令基本类似的，只是参数区别

### 配置MDS
#### 使用zpool创建mds
```bash
[root@test201 ~]# mkfs.lustre --fsname=lustrefs --mgsnode=192.168.0.201@tcp0  --mdt --index=0  /dev/sdc1

   Permanent disk data:
Target:     lustrefs:MDT0000
Index:      0
Lustre FS:  lustrefs
Mount type: ldiskfs
Flags:      0x61
              (MDT first_time update )
Persistent mount opts: user_xattr,errors=remount-ro
Parameters: mgsnode=192.168.0.201@tcp

checking for existing Lustre data: not found
device size = 20478MB
formatting backing filesystem ldiskfs on /dev/sdc1
	target name   lustrefs:MDT0000
	kilobytes     20969472
	options        -J size=819 -I 1024 -i 2560 -q -O dirdata,uninit_bg,^extents,dir_nlink,quota,huge_file,large_dir,flex_bg -E lazy_journal_init -F
mkfs_cmd = mke2fs -j -b 4096 -L lustrefs:MDT0000  -J size=819 -I 1024 -i 2560 -q -O dirdata,uninit_bg,^extents,dir_nlink,quota,huge_file,large_dir,flex_bg -E lazy_journal_init -F /dev/sdc1 20969472k
Writing CONFIGS/mountdata
```

上面的参数要指定mgs的网络，指定当前创建的是mdt，指定自己的index，也就是id号，这个是唯一值，后面就是指定的zfs的路径

#### 挂载mds的存储
这里跟上面一样，就是把服务挂载起来

准备一个挂载目录
```bash
[root@test201 ~]# mkdir /lustre/mds0
```
挂载存储
```bash
[root@test201 ~]# mount -t lustre /dev/sdc1 /lustre/mds0/
```
上面的命令执行完成以后,mds就创建完成了，mds是允许多个的这里单机节点测试一个即可

### 配置OSD
#### 配置OSD需要的底层磁盘
```bash
[root@test201 ~]# mkfs.lustre --fsname=lustrefs --mgsnode=192.168.0.201@tcp0  --ost  --index=1 /dev/sdd1

   Permanent disk data:
Target:     lustrefs:OST0001
Index:      1
Lustre FS:  lustrefs
Mount type: ldiskfs
Flags:      0x62
              (OST first_time update )
Persistent mount opts: ,errors=remount-ro
Parameters: mgsnode=192.168.0.201@tcp

checking for existing Lustre data: not found
device size = 20478MB
formatting backing filesystem ldiskfs on /dev/sdd1
	target name   lustrefs:OST0001
	kilobytes     20969472
	options        -J size=400 -I 512 -i 69905 -q -O extents,uninit_bg,dir_nlink,quota,huge_file,large_dir,flex_bg -G 256 -E resize="4290772992",lazy_journal_init -F
mkfs_cmd = mke2fs -j -b 4096 -L lustrefs:OST0001  -J size=400 -I 512 -i 69905 -q -O extents,uninit_bg,dir_nlink,quota,huge_file,large_dir,flex_bg -G 256 -E resize="4290772992",lazy_journal_init -F /dev/sdd1 20969472k
Writing CONFIGS/mountdata
```

#### 挂载osd的存储
准备一个挂载目录
```bash
[root@test201 ~]# mkdir /lustre/ost1
```
挂载存储
```bash
[root@test201 ~]# mount -t lustre /dev/sdd1 /lustre/ost1/
mount.lustre: increased '/sys/devices/pci0000:00/0000:00:10.0/host2/target2:0:3/2:0:3:0/block/sdd/queue/max_sectors_kb' from 512 to 4096
```
上面的命令执行完成以后,osd就创建完成了，下一步就是挂载客户端了

### 配置客户端
#### 挂载客户端
```bash
[root@test201 ~]# mount -t lustre 192.168.0.201@tcp0:/lustrefs /mnt
```

检查挂载
```bash
[root@test201 ~]# df -h
Filesystem                   Size  Used Avail Use% Mounted on
devtmpfs                     1.9G     0  1.9G   0% /dev
tmpfs                        1.9G     0  1.9G   0% /dev/shm
tmpfs                        1.9G   12M  1.9G   1% /run
tmpfs                        1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root       26G  4.7G   22G  19% /
/dev/sda1                   1014M  156M  859M  16% /boot
tmpfs                        378M     0  378M   0% /run/user/0
/dev/sdb1                     20G  1.2M   19G   1% /lustre/mgs
/dev/sdc1                     12G  2.4M   11G   1% /lustre/mds0
/dev/sdd1                     20G  1.3M   19G   1% /lustre/ost1
192.168.0.201@tcp:/lustrefs   20G  1.3M   19G   1% /mnt
```
检查type
```bash
[root@test201 ~]# df -T
Filesystem                  Type     1K-blocks    Used Available Use% Mounted on
devtmpfs                    devtmpfs   1917412       0   1917412   0% /dev
tmpfs                       tmpfs      1930644       0   1930644   0% /dev/shm
tmpfs                       tmpfs      1930644   12052   1918592   1% /run
tmpfs                       tmpfs      1930644       0   1930644   0% /sys/fs/cgroup
/dev/mapper/centos-root     xfs       27245572 4916572  22329000  19% /
/dev/sda1                   xfs        1038336  159252    879084  16% /boot
tmpfs                       tmpfs       386132       0    386132   0% /run/user/0
/dev/sdb1                   lustre    20304464    1176  19254816   1% /lustre/mgs
/dev/sdc1                   lustre    11599532    2376  10550232   1% /lustre/mds0
/dev/sdd1                   lustre    20200876    1240  19134780   1% /lustre/ost1
192.168.0.201@tcp:/lustrefs lustre    20200876    1240  19134780   1% /mnt
```
可以看到都是以lustre这个文件系统类型进行挂载的，现在挂载成功以后，就可以在/mnt这个挂载目录里面写数据了

lustre的检查命令
```bash
[root@test201 ~]# lfs df -h
UUID                       bytes        Used   Available Use% Mounted on
lustrefs-MDT0000_UUID       11.1G        2.3M       10.1G   1% /mnt[MDT:0]
lustrefs-OST0001_UUID       19.3G        1.2M       18.2G   1% /mnt[OST:1]

filesystem_summary:        19.3G        1.2M       18.2G   1% /mnt
```

## 总结
单节点的部署就完成了，可以看到，先是进行格式化，再挂载就启动了服务，整个部署上面是比较简单的,后面会做一些内部功能的验证
