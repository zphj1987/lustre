# lustre版本的选择

## 版本

目前正在发布的版本有:
- 2.12.x
- 2.15.x

这两个是长期支持版本，2.12版本到了2.12.8，并且即将停止支持,所以选择的是2.15.1版本，之前测试过2.12版本，稳定性存在一定的问题，触发了无故down机的问题，目前测试的2.15.1版本未出现

2.15.1版本有个支持zfs的安装bug,就是包里面找zfs模块的路径不对，并且依赖也写的不对
>https://review.whamcloud.com/c/fs/lustre-release/+/48212#message-59246f9532c815a6c6838c5d0c80b9880a74999b

这个官方已经修改，但是没有发布版本，我们自己改了一个版本，测试的时候使用这个版本就行

## 改好的版本
```bash
git clone https://e.coding.net/zphj1987/lustre-release/lustre-release.git
```
