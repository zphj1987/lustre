# osd的创建

## osd的创建
osd的创建需要指定mgs的ip地址，需要指定自己的id号


## 创建zpool
```bash
[root@lab105 ~]# zpool create -O canmount=off -o multihost=on -o cachefile=none osdpool0  xvde1
```
创建zpool可以根据需要进行定义，一般是8+2,8+3,或者其它的自定义的都行

## 格式化
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
## 挂载
```bash
[root@lab105 ~]# mkdir /lustre/osd0
[root@lab105 ~]# mount -t lustre osdpool0/osd0 /lustre/osd0
```
