# 配额说明
配额允许系统管理员限制用户、组或项目可以使用的磁盘空间数量。配额由根用户设置，可以为单个用户、组和/或项目指定。在将文件写入设置了配额的分区之前，会检查创建者组的配额。如果存在配额，则文件大小计入组的配额。如果不存在配额，则在写入文件之前检查所有者的用户配额。类似地，如果用户过度使用分配的空间，则可以控制特定函数的inode使用情况。

Lustre配额执行与标准Linux配额执行的不同有以下几个方面:
- 配额通过lfs和lctl命令管理(post-mount)。
- Lustre软件中的配额特性分布在整个系统中(因为Lustre文件系统是一个分布式文件系统)。因此，Lustre上的配额设置和行为与本地磁盘配额在以下方面有些不同:
- - 无单点管理:一些命令必须在MGS上执行，其他命令必须在mds和oss上执行，还有一些命令必须在客户端上执行。
- - 粒度:本地配额通常指定为千字节的分辨率，Lustre使用1兆字节作为最小的配额分辨率。
- - 准确性:配额信息分布在整个文件系统中，只有在静默的文件系统中才能准确计算，以便在正常使用期间最小化性能开销。
- 以量化的方式分配和使用配额。
- 客户端没有设置usrquota或grpquota选项来挂载。默认情况下是启用空间计费的，可以使用lctl conf_param在每个文件系统的基础上启用/禁用配额强制执行。

项目配额的支持需要Lustre Release 2.10或更高版本。可能需要一个打过补丁的服务器，这取决于内核版本和后端文件系统类型:
- ldiskfs with kernel version < 4.5   需要打过补丁的server
- ldiskfs with kernel version >= 4.5  不需要打过补丁的server
- zfs version >=0.8 with kernel version < 4.5 需要打过补丁的server
- zfs version >=0.8 with kernel version > 4.5 不需要打过补丁的server

lustre 2.14版本以上支持OST池配额

# 配额配置
## lctl启用磁盘配额
Lustre的配额设计将管理和执行与资源使用和计算分开。Lustre软件负责管理和执行，后端文件系统负责资源使用和计算。因此，必须现在后端磁盘系统上启用配额。由于配额设置由MGS编排，所以该节的设置命令需要在MGS节点运行。

ldiskfs后端开启有一些限制因素，包括lustre版本、e2fsprogs支持等

ZFS后端需要ZFS版本不低于0.8.0
```bash
lctl conf_param fsname.quota.ost|mdt=u|g|p|ugp|none
```
- ost——配置ost管理的块配额
- mdt——配置由mdt管理的inode配额
- u——仅对用户启用配额执行
- g——只对组启用配额执行
- p——仅对项目启用配额执行
- ugp——为所有用户、组和项目启用配额执行
- none——禁用对所有用户、组和项目的配额强制执行

例如：打开文件系统testfs1上的用户、组和项目的块配额，在MGS上执行
```bash
[root@node101 mnt]# lctl conf_param lustrefs.quota.ost=ugp
```
验证配额：
```bash
[root@node101 mnt]# lctl get_param osd-*.*.quota_slave.info
osd-ldiskfs.lustrefs-MDT0000.quota_slave.info=
target name:    lustrefs-MDT0000
pool ID:        0
type:           md
quota enabled:  none
conn to master: setup
space acct:     ug
user uptodate:  glb[0],slv[0],reint[0]
group uptodate: glb[0],slv[0],reint[0]
project uptodate: glb[0],slv[0],reint[0]
```
## lfs配额管理
配额配置支持设置软硬配额，有六种宽限期：用户块/inode软限制、用户组块/inode软限制、项目块/inode软限制。包括支持软限制和硬限制，软限制必须小于硬限制，如果设置了宽限期，则在硬限制一下可以在宽限期内超过软限制。配额不是完全准确，可能存在一些误差。配额设置执行lfs命令可以选在quota和setquota选项
- quota --显示通用配额信息（磁盘使用情况和限制）
- setquota --指定配额限制，调试宽限期。默认情况，宽限期为一周。
```bash
lfs quota [-q] [-v] [-h] [-o obd_uuid] [-u|-g|-p uname|uid|gname|gid|projid] /mountpoint
lfs quota -t {-u|-g|-p} /mount_point
lfs setquota {-u|--user|-g|--group|-p|--project} username|groupname [-b block-softlimit] \
 [-B block_hardlimit] [-i inode_softlimit] \
 [-I inode_hardlimit] /mount_point
```
限制特定目录/mnt/testfs/，特定用户配额使用
```bash
[root@node101 mnt]#  lfs setquota -u bob -b 307200 -B 309200 -i 10000 -I 11000 /mnt/testfs
[root@node101 mnt]# lfs quota -u bob -v /mnt/testfs
Disk quotas for usr bob (uid 1000):
     Filesystem  kbytes   quota   limit   grace   files   quota   limit   grace
    /mnt/testfs       0  307200  309200       -       0   10000   11000       -
lustrefs-MDT0000_UUID
                      0       -       0       -       0       -       0       -
lustrefs-MDT0001_UUID
                      0       -       0       -       0       -       0       -
lustrefs-MDT0002_UUID
                      0       -       0       -       0       -       0       -
lustrefs-OST0000_UUID
                      0       -       0       -       -       -       -       -
lustrefs-OST0001_UUID
                      0       -       0       -       -       -       -       -
Total allocated inode limit: 0, total allocated block limit: 0
```

查询执行该命令的用户及其主组的总体配额信息(磁盘使用情况和限制)，使用命令:
```bash
[root@node101 mnt]# lfs quota  /mnt/testfs
Disk quotas for usr root (uid 0):
     Filesystem  kbytes   quota   limit   grace   files   quota   limit   grace
    /mnt/testfs    7528       0       0       -     759       0       0       -
Disk quotas for grp root (gid 0):
     Filesystem  kbytes   quota   limit   grace   files   quota   limit   grace
    /mnt/testfs       0       0       0       -       0       0       0       -
```