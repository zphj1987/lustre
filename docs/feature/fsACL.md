# 访问控制列表ACL
访问控制列表ACL是用户或组对系统目录或文件的权限控制。每个对象都有一个唯一的安全属性，用于标识有权访问它的用户。ACL列出了每个用户和对象的访问权限，如读、写或执行。
Lustre文件系统的ACL控制和标准的Linux ACL类似，都是基于标准POSIX文件系统对象权限的文件条目组成，定义三类用户（所有者，组，和其他）。每个类都有一组权限关联：读（r），写（w）和执行（x）。
## ACL开启
MDS控制文件系统的元数据信息，所以需要配置MDS来启用ACL。默认在创建MDS时开启ACL
配置mds时开启acl
```bash
[root@node101 ~]# mkfs.lustre --fsname spfs --mountfsoptions=acl --mdt -mgs /dev/sda
```
或者使用mkfs.lustre命令时指定--acl选项开启
```bash
 [root@node101 ~]# mount -t lustre -o acl /dev/sda /mnt/mdt
```
在MDS上查看ACL：
```bash
[root@node101 ~]# lctl get_param -n mdc.lustrefs-MDT0000-mdc-*.connect_flags | grep acl 
acl
```
在Lustre文件系统，ACL功能属于系统范围内启用，开启是针对所有客户端。未开启ACL，客户端的ACL操作都会提示不支持的错误。

举例进行ACL相关操作：
```bash
[root@node101 mnt]# getfacl testfs/
# file: testfs/
# owner: root
# group: root
user::rwx
group::r-x
other::r-x

[root@node101 mnt]# setfacl -m user:bob:rwx testfs/
[root@node101 mnt]# getfacl testfs/
# file: testfs/
# owner: root
# group: root
user::rwx
user:bob:rwx
group::r-x
mask::rwx
other::r-x
```