* lustre文件系统的架构
 ** [lustre的核心组件](architecture/core)
 ** [lustre最小化部署](architecture/smallinstall)
 ** [lustre的架构体系](architecture/lustrearch)
 ** [openzfs和ldiskfs的区别](architecture/lfsvszfs)

* lustre的基本安装
 ** [lustre的操作系统的选择](install/os)
 ** [lustre的版本的选择](install/lustreversion)
 ** [lustre的打包编译(lustre内核)](install/buildrpm)
 ** [lustre的打包编译(系统内核)](install/buildrpmoskernel)
 ** [lustre的包安装](install/installrpm)

* lustre的基本配置
 ** [mgs的创建](configure/mgscreate)
 ** [mds的创建](configure/mdscreate)
 ** [osd的创建](configure/osdcreate)
 ** [客户端挂载](configure/clientcreate)
 ** [mds的删除](configure/mdsremove)
 ** [osd的删除](configure/osdremove)
 ** [mirror的配置](configure/mirrorconfig)

* lustre的功能特性分析
 ** [lustre读写IO过程](sourcecode/IOprocess)
 ** [lustre多客户一致性保证](sourcecode/clientcache)
 ** [lustre写入的文件是如何分布的](feature/filelayout)
 ** [lustre权限控制ACL](feature/fsACL)
 ** [lustre配额管理](feature/quota)
 ** [lustre基于ZFS的快照](feature/ZFS-snapshot)
 ** [mirror功能的分析]()
 ** [lustre如果做了mirror如何分布的]()
 ** [lustre坏了osd后，如何保证继续使用]()

* lustre的常规运维操作
 ** [mirror损坏一个ost后如何处理]()
 ** [lustre的zfs出现降级后如何处理]()
 ** [mgs的备份恢复]()
 ** [mds的备份恢复]()

