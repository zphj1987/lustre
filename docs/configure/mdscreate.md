# mds的创建

## mds的创建 
mds的创建需要指定自己的id号，也要指定mgs的ip地址

## 创建zpool
```bash
[root@lab105 ~]# zpool create -O canmount=off -o multihost=on -o cachefile=none mdspool  xvdc1
```
zpool的创建也是需要给mds加入一个冗余的，这个在zpool层面做控制即可

## 格式化
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

## 挂载服务

```bash
[root@lab105 ~]# mkdir /lustre/mds0 -p
[root@lab105 ~]# mount -t lustre mdspool/mds0 /lustre/mds0
```

挂载就等于启动了服务了
