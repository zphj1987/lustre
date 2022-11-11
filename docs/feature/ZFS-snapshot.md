
# ZFS快照
快照提供了从以前创建的检查点快速恢复文件的功能，而无需求助于脱机备份或远程副本。快照还提供了一种版本控制存储的方法，可以用于恢复丢失的文件或文件的以前版本。

文件系统快照旨在挂载在用户可访问的节点上，例如登录节点，这样用户就可以在不需要管理员干预的情况下恢复文件(例如在意外删除或覆盖之后)。当用户访问快照文件系统时，可以通过自动挂载的方式挂载快照文件系统，而不是挂载所有快照，以在快照不使用时减少登录节点的开销。

从快照中恢复丢失的文件通常比从任何脱机备份或远程副本中恢复要快得多。但是，请注意快照不会提高存储的可靠性，而且与任何其他存储卷一样容易出现硬件故障。

## 快照支持要求
- Lustre版本要求：2.10以上
- MGS节点需要配置到其他服务节点免密ssh登录
- 该特性默认启用，不能禁用
- 快照的管理是通过MGS上的lctl命令完成
- Lustre快照基于Copy-On-Write

## 快照配置
快照工具从MGS上的/etc/ldev.conf文件加载系统配置，并调用相关的ZFS命令来维护所有目标上的Lustre快照片段(MGS/MDS/OST)。需要先对/etc/ldev.conf配置文件进行文件系统的相关配置，例如：
```bash
 cat /etc/ldev.conf
2 host-mdt1 - myfs-MDT0000 zfs:/tmp/myfs-mdt1/mdt1
3 host-mdt2 - myfs-MDT0001 zfs:myfs-mdt2/mdt2
4 host-ost1 - OST0000 zfs:/tmp/myfs-ost1/ost1
5 host-ost2 - OST0001 zfs:myfs-ost2/ost2
```
### 创建快照
在MGS上运行lctl命令，为已有的Lustre文件系统创建快照。
```bash
lctl snapshot_create [-b | --barrier [on | off]] [-c | --comment
comment] -F | --fsname fsname> [-h | --help] -n | --name ssname>
[-r | --rsh remote_shell][-t | --timeout timeout]
```
参数说明：
- -b 创建快照之前设置写屏障。默认为on
- -c 快照目的说明
- -F 文件系统名
- -h 帮助信息
- -n 快照名称
- -r 用于与远程目标进行通信的远程shell。默认ssh
- -t 写屏障的时间周期
### 删除快照
在MGS上使用lctl命令删除已有的快照。
```bash
lctl snapshot_destroy [-f | --force] <-F | --fsname fsname>
<-n | --name ssname> [-r | --rsh remote_shell]
```
参数说明：
- -f 强制销毁快照
- -F 文件系统名
- -h 帮助信息
- -n 快照名称
- -r 用于与远程目标进行通信的远程shell。默认ssh
### 挂载快照
快照被视为独立的文件系统，可以挂载在Lustre客户机上。快照文件系统必须使用-o ro选项挂载为只读文件系统。如果mount命令中不包含只读选项，会挂载失败。在客户机上挂载快照之前，必须首先使用lctl在服务器上挂载快照。
```bash
lctl snapshot_mount <-F | --fsname fsname> [-h | --help]
<-n | --name ssname> [-r | --rsh remote_shell]
```
参数说明：
- -F 文件系统名
- -h 帮助信息
- -n 快照名称
- -r 用于与远程目标进行通信的远程shell。默认ssh

在服务器上成功地挂载快照之后，客户机现在可以将快照作为只读文件系统挂载。例如：
```bash
#服务器挂载快照
mgs# lctl snapshot_mount -F myfs -n snapshot_20170602
#使用lctl snapshot_list获取快照本身的fsname
mgs# ss_fsname=$(lctl snapshot_list -F myfs -n snapshot_20170602 |
 awk '/^snapshot_fsname/ { print $2 }')
 #最后客户端挂载快照
 client# mount -t lustre -o ro $MGS_nid:/$ss_fsname $local_mount_point
```
### 卸载快照
要从服务器卸载快照，首先要在每个客户端上使用标准umount命令从所有客户端卸载快照文件系统。在所有客户端卸载快照文件系统后，在挂载快照的服务器节点上运行以下lctl命令:
```bash
lctl snapshot_umount [-F | --fsname fsname] [-h | --help]
<-n | -- name ssname> [-r | --rsh remote_shell]
```
参数说明：
- -F 文件系统名
- -h 帮助信息
- -n 快照名称
- -r 用于与远程目标进行通信的远程shell。默认ssh

例如：lctl snapshot_umount -F myfs -n snapshot_20170602
### 列举快照
在MGS上使用以下lctl命令列出给定文件系统的可用快照:
```bash
lctl snapshot_list [-d | --detail] <-F | --fsname fsname>
[-h | -- help] [-n | --name ssname] [-r | --rsh remote_shell]
```
参数说明：
- -d 列出指定快照的详细信息
- -F 文件系统名
- -h 帮助信息
- -n 快照名称。如果没有指定快照名，将显示此文件系统的所有快照
- -r 用于与远程目标进行通信的远程shell。默认ssh
### 修改快照属性
Lustre快照目前有5个用户可见属性;快照名称、快照注释、创建时间、修改时间、快照文件系统名称。其中，前两个属性可以修改。重命名遵循一般的ZFS快照名称规则，例如最大长度为256字节，不能与保留名称冲突，等等。
在MGS上使用lctl命令修改快照属性:
```bash
lctl snapshot_modify [-c | --comment comment]
<-F | --fsname fsname> [-h | --help] <-n | --name ssname>
[-N | --new new_ssname] [-r | --rsh remote_shell]
```
参数说明：
- -c 更新快照说明
- -F 文件系统名
- -h 帮助信息
- -n 快照名称
- -N 重新命名快照为new_ssname
- -r 用于与远程目标进行通信的远程shell。默认ssh
## 快照全局写屏障
快照是跨多个MDT和OST的非原子快照，这意味着如果在执行快照时文件系统上有活动，那么在MDT和OST快照之间的间隔内创建或销毁的文件可能存在用户可见的名称空间不一致。为了创建文件系统的一致快照，我们可以设置全局写障碍，或“冻结”系统。一旦设置，所有元数据修改都将被阻塞，直到写障碍被主动移除(“解冻”)或过期。用户可以在全局障碍上设置超时参数，也可以显式删除障碍。缺省超时时间为30秒。
重要的是要注意快照是可以在没有全局障碍的情况下使用的。如果没有使用屏障，只有客户端当前正在修改的文件(写、创建、断开链接)可能不一致。其他当前未被修改的文件即使没有这个障碍也可以使用。
当使用lctl snapshot_create的- b选项请求时，snapshot create命令将在内部调用写屏障。因此，在使用快照时不需要显式地使用该屏障，而是在这里作为创建快照之前使文件系统安静下来的选项。
默认是开启状态，不需要额外操作
 
## 快照日志
有快照活动的日志可以在以下文件中找到:/var/log/ lsnapshot.log。该文件包含关于快照创建时间、更改属性、挂载时间以及其他快照信息的信息。
/var/log/lsnapshot文件示例如下:
```bash
Mon Mar 21 19:43:06 2016
(15826:jt_snapshot_create:1138:scratch:ssh): Create snapshot lss_0_0
successfully with comment <(null)>, barrier <enable>, timeout <30>
Mon Mar 21 19:43:11 2016(13030:jt_snapshot_create:1138:scratch:ssh):
Create snapshot lss_0_1 successfully with comment <(null)>, barrier
<disable>, timeout <-1>
Mon Mar 21 19:44:38 2016 (17161:jt_snapshot_mount:2013:scratch:ssh):
The snapshot lss_1a_0 is mounted
Mon Mar 21 19:44:46 2016
(17662:jt_snapshot_umount:2167:scratch:ssh): the snapshot lss_1a_0
have been umounted
Mon Mar 21 19:47:12 2016
(20897:jt_snapshot_destroy:1312:scratch:ssh): Destroy snapshot
lss_2_0 successfully with force <disable>
```
