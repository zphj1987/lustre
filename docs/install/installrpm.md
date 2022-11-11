# lustre的包安装

## 配置源
```bash
[root@lab110 ~]# cat /etc/yum.repos.d/lustre.repo
[lustre-server]
name=lustre-server
baseurl=https://downloads.whamcloud.com/public/lustre/lustre-2.15.1/el8.6/server/
gpgcheck=0
enable=1
```
powertool和epel的源也都开启一下

配置好源,然后安装内核
```
yum install kernel-4.18.0-372.9.1.el8_lustre.x86_64 kernel-devel-4.18.0-372.9.1.el8_lustre.x86_64
```
需要重启下确认下内核变了

## 安装依赖包
```bash
yum install zfs zfs-dkms
yum install expect libnl3-devel libyaml-devel python2  flex bison libmount-devel python36 python36-devel 
```

## 安装lustre的包
```bash
rpm -ivh lustre-zfs-dkms-2.15.1-1.el8.noarch.rpm
rpm -ivh lustre-2.15.1-1.el8.x86_64.rpm  lustre-osd-zfs-mount-2.15.1-1.el8.x86_64.rpm lustre-zfs-dkms-2.15.1-1.el8.noarch.rpm lustre-iokit-2.15.1-1.el8.x86_64.rpm
```
## 网络配置一下
```bash
echo 'options lnet networks=tcp0(ens33)' > /etc/modprobe.d/lustre.conf
```
安装完成可以开始进行其它相关的配置测试工作了

