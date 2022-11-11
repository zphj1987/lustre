# lustre的最小化部署

## 最小化部署

我们在接触一个系统前，如果对这个系统不是很了解的情况下，会觉得系统非常复杂，很多时候其实是文档配套太差

即时是像lustre这样的系统，发布出来的文档几百页，这个看完还不清楚从哪里下手，我像看下这个系统是怎么样的，没有一个简单的入门的

很多新系统的都会有一个简单的部署指南，可能就几分钟就能部署起来一个能够使用的系统，本章节就是总结这个事情的

我们部署一个最小系统，包的安装部分在其它章节讲，本篇的前提是包都安装好的情况下，一步步配置一个可以使用的系统，分下面的几个步骤

因为现在都是通过zfs的底层来实现成本的降低，所以本文档的操作都是以zfs为底层的部署的

- 1、部署mgs
- 2、部署mds
- 3、部署osd
- 4、部署client
- 5、写数据

## 部署系统

### 准备硬件环境
安装一台centos8的操作系统，更新好了内核（zfs底层的不用更新），zfs安装好了，相关软件包安装好了，只用进行相关的配置
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

#### 配置MGS需要的底层磁盘
xvdb1准备给mgs使用,使用这个盘创建zpool
```bash
zpool create -O canmount=off -o multihost=on -o cachefile=none mgspool  xvdb1 
```
上面有几个参数是用来屏蔽掉zfs的一些默认的挂载，我们需要使用lustre来挂载这些pool

检查创建的zpool
```bash
[root@lab105 ~]# zpool list
NAME      SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
mgspool  99.5G   105K  99.5G        -         -     0%     0%  1.00x    ONLINE  -
```

#### 使用zpool创建mgs
```bash
[root@lab105 ~]# mkfs.lustre --fsname=lustrefs --mgs --backfstype=zfs  mgspool/mgt

   Permanent disk data:
Target:     MGS
Index:      unassigned
Lustre FS:  lustrefs
Mount type: zfs
Flags:      0x64
              (MGS first_time update )
Persistent mount opts: 
Parameters:
checking for existing Lustre data: not found
mkfs_cmd = zfs create -o canmount=off  mgspool/mgt
  xattr=sa
  dnodesize=auto
Writing mgspool/mgt properties
  lustre:version=1
  lustre:flags=100
  lustre:index=65535
  lustre:fsname=lustrefs
  lustre:svname=MGS

```
这个地方指定了lustrefs，后面创建其它的也要指定，这里是单机的，就没有指定备用节点的一些信息，后面指定了mgs的pool名称，也就是上面我们创建的pool名称，然后后面再指定一个路径


#### 挂载mgs的存储
lustre和其它存储有个不同的地方就是，服务的启动是通过挂载来提供的，也就是挂载后，服务对外自动提供，卸载后，这个服务就自动停止，所以这里的挂载等同于启动服务

准备一个挂载目录
```bash
[root@lab105 ~]# mkdir /lustre/mgs -p
```
挂载存储
```bash
[root@lab105 ~]# mount -t lustre mgspool/mgt /lustre/mgs/
```
指定完上面的，mgs的服务就创建完毕了,后面创建mds和osd的流程和命令基本类似的，只是参数区别


### 配置MDS
#### 配置MDS需要的底层磁盘
xvdc1准备给mgs使用,使用这个盘创建zpool
```bash
zpool create -O canmount=off -o multihost=on -o cachefile=none mdspool  xvdc1
```
上面有几个参数是用来屏蔽掉zfs的一些默认的挂载，我们需要使用lustre来挂载这些pool

检查创建的zpool
```bash
[root@lab105 ~]# zpool list
NAME      SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
mdspool  99.5G   152K  99.5G        -         -     0%     0%  1.00x    ONLINE  -
mgspool  99.5G  2.83M  99.5G        -         -     0%     0%  1.00x    ONLINE  -
```

#### 使用zpool创建mds
```bash
[root@lab105 ~]# mkfs.lustre --fsname=lustrefs --mgsnode=192.168.50.105@tcp0  --mdt --index=0   --backfstype=zfs  mdspool/mds0

   Permanent disk data:
Target:     lustrefs:MDT0000
Index:      0
Lustre FS:  lustrefs
Mount type: zfs
Flags:      0x61
              (MDT first_time update )
Persistent mount opts: 
Parameters: mgsnode=192.168.50.105@tcp
checking for existing Lustre data: not found
mkfs_cmd = zfs create -o canmount=off  mdspool/mds0
  xattr=sa
  dnodesize=auto
Writing mdspool/mds0 properties
  lustre:mgsnode=192.168.50.105@tcp
  lustre:version=1
  lustre:flags=97
  lustre:index=0
  lustre:fsname=lustrefs
  lustre:svname=lustrefs:MDT0000
```
上面的参数要指定mgs的网络，指定当前创建的是mdt，指定自己的index，也就是id号，这个是唯一值，后面就是指定的zfs的路径

#### 挂载mds的存储
这里跟上面一样，就是把服务挂载起来

准备一个挂载目录
```bash
[root@lab105 ~]# mkdir /lustre/mds0 -p
```
挂载存储
```bash
[root@lab105 ~]# mount -t lustre mdspool/mds0 /lustre/mds0
```
上面的命令执行完成以后,mds就创建完成了，mds是允许多个的这里单机节点测试一个即可

### 配置OSD
#### 配置OSD需要的底层磁盘
xvde1准备给mgs使用,使用这个盘创建zpool
```bash
zpool create -O canmount=off -o multihost=on -o cachefile=none osdpool0  xvde1
```
上面有几个参数是用来屏蔽掉zfs的一些默认的挂载，我们需要使用lustre来挂载这些pool

检查创建的zpool
```bash
[root@lab105 ~]# zpool list
NAME       SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
mdspool   99.5G  3.17M  99.5G        -         -     0%     0%  1.00x    ONLINE  -
mgspool   99.5G  2.86M  99.5G        -         -     0%     0%  1.00x    ONLINE  -
osdpool0  99.5G   106K  99.5G        -         -     0%     0%  1.00x    ONLINE  -
```

#### 使用zpool创建osd
```bash
[root@lab105 ~]# mkfs.lustre  --fsname=lustrefs  --mgsnode=192.168.50.105@tcp0  --ost --index=0 --backfstype=zfs  osdpool0/osd0

   Permanent disk data:
Target:     lustrefs:OST0000
Index:      0
Lustre FS:  lustrefs
Mount type: zfs
Flags:      0x62
              (OST first_time update )
Persistent mount opts: 
Parameters: mgsnode=192.168.50.105@tcp
checking for existing Lustre data: not found
mkfs_cmd = zfs create -o canmount=off  osdpool0/osd0
  xattr=sa
  dnodesize=auto
  recordsize=1M
Writing osdpool0/osd0 properties
  lustre:mgsnode=192.168.50.105@tcp
  lustre:version=1
  lustre:flags=98
  lustre:index=0
  lustre:fsname=lustrefs
  lustre:svname=lustrefs:OST0000
```


#### 挂载osd的存储
准备一个挂载目录
```bash
[root@lab105 ~]# mkdir /lustre/osd0 -p
```
挂载存储
```bash
[root@lab105 ~]# mount -t lustre osdpool0/osd0 /lustre/osd0
```
上面的命令执行完成以后,osd就创建完成了，下一步就是挂载客户端了

### 配置客户端
#### 挂载客户端
```bash
[root@lab105 ~]# mount -t lustre 192.168.50.105@tcp0:/lustrefs /mnt
```

检查挂载
```bash
[root@lab105 ~]# df -h
Filesystem                    Size  Used Avail Use% Mounted on
devtmpfs                      1.8G     0  1.8G   0% /dev
tmpfs                         1.9G     0  1.9G   0% /dev/shm
/dev/xvda2                    117G  4.1G  112G   4% /
mgspool/mgt                    96G  2.7M   96G   1% /lustre/mgs
mdspool/mds0                   96G  3.0M   96G   1% /lustre/mds0
osdpool0/osd0                  96G  3.0M   96G   1% /lustre/osd0
192.168.50.105@tcp:/lustrefs   96G  3.0M   96G   1% /mnt
```
检查type
```bash
[root@lab105 ~]# df -T
Filesystem                   Type     1K-blocks    Used Available Use% Mounted on
devtmpfs                     devtmpfs   1883076       0   1883076   0% /dev
tmpfs                        tmpfs      1899928       0   1899928   0% /dev/shm
/dev/xvda2                   xfs      121640928 4221904 117419024   4% /
mgspool/mgt                  lustre   100299392    2688 100294656   1% /lustre/mgs
mdspool/mds0                 lustre   100299392    3072 100294272   1% /lustre/mds0
osdpool0/osd0                lustre   100298752    3072 100293632   1% /lustre/osd0
192.168.50.105@tcp:/lustrefs lustre   100298752    3072 100293632   1% /mnt
```
可以看到都是以lustre这个文件系统类型进行挂载的，现在挂载成功以后，就可以在/mnt这个挂载目录里面写数据了

lustre的检查命令
```bash
[root@lab105 ~]# lfs df -h
UUID                       bytes        Used   Available Use% Mounted on
lustrefs-MDT0000_UUID       95.7G        3.0M       95.6G   1% /mnt[MDT:0]
lustrefs-OST0000_UUID       95.7G        3.0M       95.6G   1% /mnt[OST:0]

filesystem_summary:        95.7G        3.0M       95.6G   1% /mnt
```

## 总结
单节点的部署就完成了，可以看到，都是先在底层创建一个zpool的存储，然后在创建lustre的角色的时候指定到zpool的一个路径，就可以格式化完成，再挂载就启动了服务，整个部署上面是比较简单的,后面会做一些内部功能的验证
