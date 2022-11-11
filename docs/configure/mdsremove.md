# mds删除(TODO)

## mds删除
mds的删除是在多mds的情况下进行的删除的操作

## 构建一个多mds的环境
```bash
[root@lab105 ~]# zpool create -O canmount=off -o multihost=on -o cachefile=none mdspool1  xvdf1
[root@lab105 ~]# mkfs.lustre --fsname=lustrefs --mgsnode=192.168.50.105@tcp0  --mdt --index=1   --backfstype=zfs  mdspool1/mds1
[root@lab105 ~]# mkdir /lustre/mds1
[root@lab105 ~]# mount -t lustre mdspool1/mds1 /lustre/mds1
```
## 检查mds的占用
```bash
[root@lab105 ~]# lfs df
UUID                   1K-blocks        Used   Available Use% Mounted on
lustrefs-MDT0000_UUID   100299392        3072   100294272   1% /mnt[MDT:0]
lustrefs-MDT0001_UUID   100299392        2944   100294400   1% /mnt[MDT:1]
lustrefs-OST0000_UUID   100298752        3072   100293632   1% /mnt[OST:0]

filesystem_summary:    100298752        3072   100293632   1% /mnt
```

## 目录与mds的关系
通过写入数据可以看到，mds的目录与mds的关系是随机分配的，子目录并不是完全继承父目录的绑定关系的，这样能够做到基本完全随机的情况

```bash
[root@lab105 ~]#  lfs getstripe --mdt-index   /mnt/testdir1/
0
[root@lab105 ~]#  lfs getstripe --mdt-index   /mnt/testdir2/
1
```
查看目录与mds的绑定关系,返回的是mds的索引

也可以使用内部命令进行处理
```bash
[root@lab105 ~]# lfs find --mdt lustrefs-MDT0001 /mnt |head -n 5
/mnt/testdir1/dir2
/mnt/testdir1/dir2/b6985
/mnt/testdir1/dir2/b8890
/mnt/testdir1/dir2/b6811
/mnt/testdir1/dir2/b8904
[root@lab105 ~]# lfs find --mdt lustrefs-MDT0000 /mnt |lfs migrate -m 0 
/mnt
/mnt/testdir1
/mnt/testdir1/b2228
/mnt/testdir1/a3898
/mnt/testdir1/a3556
```

lfs find --ost ost_name /mount/point | lfs_migrate -y

遍历查找目录的对应mds
```bash
[root@lab105 ~]# find /mnt/ -type d -exec  lfs getdirstripe -m    {} \; 
0
0
0
1
1
```



## 分析
目前掌握的情况来看，mdt0也就是根目录绑定的mds是无法删除的，其它的mds可以通过提前把数据转移走，然后做维护操作来实现删除，那么这个地方，备份相关的工作就需要做好，元数据的数据量不大，并且本身是有磁盘级别的保护的，再多一层元数据的备份，这个是非常有必要的，这个在实际生产环境当中也是需要这么去做的

## 删除非主mds的方法









