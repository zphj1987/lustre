# lustre服务线程调优
oss/mds的IO服务线程定义，每个服务线程消耗1.5MB的内存资源
## oss服务相关参数
- {service}.threads_min  oss启动后最⼩的服务线程数
- {service}.threads_max oss启动后最⼤的服务线程数
- {service}.threads_started oss启动后已经启动的服务线程数
list相关参数
```bash 
lctl list_param *.*.*.* |grep threads_*|grep ost 
```
set相关参数
```bash 
 lctl set_param ost.OSS.ost.threads_max 65
```
get相关参数
```bash
 lctl get_param ost.OSS.ost.threads_max
```  

## mds服务相关参数
（1）mds服务相关线程参数
- {service}.threads_min  mds启动后最⼩的服务线程数
- {service}.threads_max mds启动后最⼤的服务线程数
- {service}.threads_started mds启动后已经启动的服务线程数
list相关参数
```bash
lctl list_param *.*.*.* |grep threads_*|grep mds
```
（2）mds绑定cpu核心调优
mds服务线程绑定CPU核可以提高缓存命中和充分利用内存的局限性
语法：
```bash
options mdt mds_num_cpts=[0]  #绑定在cpu0上 
options mds_rdpg_num_cpts=[0-1] #read_page服务线程绑定在cpu0~cpu1上 
options lnet networks=tcp0(eth0) #绑定⽹卡eth0到tcp0
```
设置方法：
在/etc/modprobe.d/lustre.conf⽂件中添加⼀⾏:options mdt mds_num_cpts=[0-6],含义是mds的线程绑定在cpu0~cpu5

# lustre网络调优
（1）LNet缓冲区调优
⽹络传输和接送缓冲，options ksocklnd tx_buffer_size 和rx_buffer_size都设置为0，lustre会⾃动调整缓冲区⼤⼩。                                                   
设置：
在/etc/modprobe.d/lustre.conf⽂件中append⼀⾏:options ksocklnd tx_buffer_size=0 rx_buffer_size=0

（2）LNet服务绑定CPU
LNet服务绑定到⼀个或者多个CPU上，所有的消息处理线程都会在这些 CPU上，提高处理效率
```bash
options lnet networks=tcp0(enp0s5)
options tcp0(enp0s5)[0]/options tcp0(enp0s5)[0,1] #前者绑定在cpu0上，后者绑定到cpu0和cpu1上
```
设置：在/etc/modprobe.d/lustre.conf⽂件中append⼀⾏:options tcp0(enp0s5)[0,1]

(3) LNet包分发和处理
默认情况LNet分发消息到CPU核处理根据nid的哈希，存在所有消息可能被同⼀个cpu核处理，这样就降低了处理消息效率，可以设置消息分发和处理按照round- robin⽅式提⾼消息处理效率。主要支持一下几种模式：
OFF-禁⽤round-robin模式 
ON-开启RR模式
RR_RT-开启路由消息的RR模式 
HASH_RT-按照源端NID哈希分发消息
设置：
```bash
$ lctl set_param portal_rotor 
portal_rotor= { portals: all rotor: ON description: round-robin dispatch all PUT messages for wildcard portals }
```

# IO块大小设置
后端zfs-osd/ldiskfs-osd之间的的IO最⼤不能超过obdfilter.{fsname}-OST000{index}.brw_ size。客户端的参数osc.*.max_pages_per_rpc控制客户端RPC⼤⼩，这个参数永远⼩于等于 obdfilter.{fsname}-OST000{index}.brw_size，当客户端连接到OST,客户端会读取brw_size然后设置⾃身的max_pages_per_rpc的RPC⼤⼩

osd端设置
```bash
$ lctl list_param *.*.* |grep brw_size obdfilter.bigfs-OST0001.brw_size obdfilter.bigfs-OST0002.brw_size #这⾥默认是4M 
$ lctl get_param obdfilter.bigfs-OST0001.brw_size obdfilter.bigfs-OST0001.brw_size=4 
#lctl set_param 添加参数-P 会持久化这个参数配置 
$ lctl set_param -P obdfilter.bigfs-OST0001.brw_size=16M 
$ lctl set_param -P obdfilter.bigfs-OST0002.brw_size=16M 
# 设置后⽣效，但是针对新连接到的客户端⽣效，已经连接客户端需要重新设置 
$ lctl set_param obdfilter.bigfs-OST0001.brw_size obdfilter.bigfs-OST0001.brw_size=16
```

client端设置
```bash
 #查看mdc的max_pages_per_rpc，这⾥参数单位是字节，每个page默认是4k, 256*4*1024=1M 
$ lctl get_param mdc.bigfs-MDT0000-mdc-ffff995143192000.max_pages_per_rpc mdc.bigfs-MDT0000-mdc-ffff995143192000.max_pages_per_rpc=256 
# 客户端请求OST的rpc⼤⼩是 1024*4*1024 = 4M，和后端brw_size保持⼀致 
$ lctl get_param osc.bigfs-OST0001-osc-ffff995143192000.max_pages_per_rpc osc.bigfs-OST0001-osc-ffff995143192000.max_pages_per_rpc=1024
# 设置为4096
$ lctl set_param osc.bigfs-OST0001-osc-ffff9951567b9000.max_pages_per_rpc osc.bigfs-OST0001-osc-ffff9951567b9000.max_pages_per_rpc=4096
```

