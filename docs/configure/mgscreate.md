# mgs的创建

## mgs的介绍
mgs是管理整个集群的配置的上线和下线的，如果不涉及到上下线的操作的时候，mgs的服务上下线并不会影响到业务
而mgs又是整个集群的信息维护的，而lustre本身不支持多mgs，只能主备的关系，并且底层数据要一致，所以建议是对mgs的配置默认都需要进行备份的

## mgs的创建
mgs和其它的存储角色一样，都是需要一个底层数据支撑的，所以都是需要先创建一个zpool，再格式化，再挂载的

### 创建zpool
```bash
[root@lab105 ~]# zpool create -O canmount=off -o multihost=on -o cachefile=none mgspool  xvdb1
```
上面的是单盘的zpool，实际使用过程中应该是需要使用mirror的，底层给磁盘留冗余

### 格式化

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

### 挂载服务
```bash
[root@lab105 ~]# mkdir /lustre/mgs -p
[root@lab105 ~]# mount -t lustre mgspool/mgt /lustre/mgs/
```
上面的操作完成后，mgs就配置好了
