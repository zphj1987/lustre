# os 发布记录

## 2022-12-01 16:00
AlmaLinux-8-6-x86_64-dvd-lustre-kernel-12011514.iso
版本说明:
1、增加kernel-debug包到内核（ludiskfs-dkms需要的包）
2、增加e2fsprogs包到iso（ludiskfs的补丁包）
3、修改默认安装模式为Server（安装界面不用动模式了）
4、提供zfs和ludisk双兼容的相关的包，简化流程
   zfs 安装6六个包 （先安装）
   lustre 安装5个包




## 2022-11-30 17:55
AlmaLinux-8-6-x86_64-dvd-lustre-kernel-11301755.iso
版本说明:
1、加入develop tool，减少后续安装（无需勾选）
2、zfs相关的依赖包默认安装:
   - expect
   - libyaml-devel 
   - libnl3-devel 
   - python2 
   - libmount-devel
   - libuuid-devel  
   - libblkid-devel
   - perl
   - dkms
   - sysstat
   上述的包全部放入到develop group里面了
3、内核kernel-devel包默认安装

使用方法:
   1、安装操作系统，选择server
   2、zfs的包安装
    - libnvpair3-2.1.2-1.el8.x86_64.rpm  
    - libzfs5-2.1.2-1.el8.x86_64.rpm    
    - zfs-2.1.2-1.el8.x86_64.rpm
    - libuutil3-2.1.2-1.el8.x86_64.rpm   
    - libzpool5-2.1.2-1.el8.x86_64.rpm  
    - zfs-dkms-2.1.2-1.el8.noarch.rpm
   3、lustre的包安装
   - lustre-zfs-dkms-2.15.1-1.el8.noarch.rpm 
   - lustre-2.15.1-1.el8.x86_64.rpm  
   - lustre-osd-zfs-mount-2.15.1-1.el8.x86_64.rpm 
   - lustre-iokit-2.15.1-1.el8.x86_64.rpm

