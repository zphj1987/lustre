# lustre的打包编译(lustre内核)

## 编译相关
因为官方的包存在安装的问题，所以需要自己打包，本文档就是详细记录打包的步骤的,需要合入官方的patch

## 处理方法
### 添加源
```bash
[root@lab110 ~]# cat /etc/yum.repos.d/lustre.repo
[lustre-server]
name=lustre-server
baseurl=https://downloads.whamcloud.com/public/lustre/lustre-2.15.1/el8.6/server/
gpgcheck=0
enable=1
```

### 安装内核
```bash
yum install kernel-4.18.0-372.9.1.el8_lustre.x86_64 kernel-devel-4.18.0-372.9.1.el8_lustre.x86_64
```
这里说明一下，dkms是动态编译内核，也就是会自动去根据当前使用的内核版本进行模块的编译，ldiskfs底层是需要修改带lustre的patch的内核的，而zfs的底层是不需要打patch的内核的，客户端是不需要打patch的内核的

总结一下也就是：

如果使用的是openzfs的底层，内核是没有约束的，只需要看是不是编译能通过即可,打包的时候会根据当前打包环境生成kmod的模块，而安装的时候有两种方式，kmod方式和dkms的方式，kmod需要限定内核的，dkms不限定内核

基于上面的我们目前做一下约束:
- 1、打包使用lustre官方提供的内核(后续如果需要kmod安装方式，再切换我们自己的内核环境打包,或者使用os自带内核打包)
- 2、运行环境server我们也都使用lustre官方提供的内核（有特殊需求再改）
- 3、运行环境client我们也都使用lustre官方提供的内核（有特殊需求再改）
- 4、目前都以dkms方式安装

后面会修改一个版本支持默认系统内核(TODO)

### 下载lustre源码
```bash
https://downloads.whamcloud.com/public/lustre/latest-release/el8.6/server/SRPMS/lustre-2.15.1-1.src.rpm
rpm2cpio lustre-2.15.1-1.src.rpm |cpio -div
yum -y groupinstall 'Development Tools'
yum install dkms libmount-devel libnl3-devel libyaml-devel kernel-rpm-macros 	kernel-abi-whitelists
yum install -y zfs libzfs5-devel zfs-dkms 
yum install -y  python36 python36-devel
```

#### 解压源码
```bash
tar -xvf lustre-2.15.1.tar.gz
```

修改patch
官方提供的修改patch
> https://review.whamcloud.com/c/fs/lustre-release/+/48212#message-59246f9532c815a6c6838c5d0c80b9880a74999b

改lustre-dkms_pre-build.sh这个的时候，增加
```bash
--without-spl
```
因为spl现在在zfs2.12版本已经集成进去了，不需要再单独引入源码编译了，直接禁用即可

### 开始编译
```bash
sh ./autogen.sh
./configure --disable-ldiskfs
make rpms
```
这里disable-ldiskfs就是开启zfs了，后续如果需要版本支持ldiskfs，再开启即可，如果同时维护两套底层，整体会很复杂很多，这个可以先支持一种，后面再加入支持即可
编译完成以后就会生成rpm包

还有个zfs的包需要编译下
```bash
rpmbuild -bb --with zfs lustre-dkms.spec
cp -ra  /root/rpmbuild/RPMS/noarch/lustre-zfs-dkms-2.15.1-1.el8.noarch.rpm /root/lustre-release/
```
上面的编译完成以后相关的包就生成了

因为服务端的包，包含了客户端的包，所以目前测试我们都直接安装服务端的包即可，后面再说明不同的安装方式:
- 客户端和服务端都安装
- 只装客户端
