# 操作系统的选择


## 操作系统
因为lustre对外发布的包是el8的，也就是centos8系列的，但是centos8到centos8.5就不再发布新版本了，而是以centos stream方式发布，也就是滚动发行

而因为开发周期的原因，需要选择一个比较稳定的版本会比较好，也就是会自己发布8.6这种版本的

从公网的包提供平台可以看到:

https://pkgs.org/download/vim

centos8的改版里面
```bash
AlmaLinux 8
Rocky Linux 8
```
是在平台上面有包的，也就是具备一定的维护能力的，而AlmaLinux背后是有商业支持的，所以目前测试采用的这个版本

后续如果存在问题，再迁移到其它平台，整体动作也不大，只要是centos8改的os，基本都是兼容的

