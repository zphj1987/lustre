# os 发布记录

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

