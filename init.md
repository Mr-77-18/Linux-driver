## 环境搭建
目标：
1. 通过nfs过载跟文件系统，实现快速开发调试
2. 通过tftp烧写zImage , dtb

搭建成功后的环境如下：

ubuntu中：

ip:172.31.225.188

目录：
1. /home/lsh/linuxdriver/nfs/
2. /home/lsh/linuxdriver/tftpboot/

uboot参数设置：

```c

setenv bootcmd 'tftp 80800000 zImage; tftp 83000000 imx6ull-alientek-emmc.dtb; bootz 80800000 - 83000000'

setenv bootargs 'console=ttymxc0# 15200 root=/dev/nfs rw nfsroot=172.31.225.188:/home/lsh/linux/nfs/rootfs,proto=tcp rw ip=172.31.225.100:172.31.225.188:172.31.225.1:255.255.255.0::eth0:off'

`````

#### 步骤
1. ubuntu中准备好目录
创建目录/home/lsh/linuxdriver/nfs/rootfs/, /home/lsh/linuxdirver/tftpboot/。可以根据自己的实际情况替换/home/lsh/linuxdriver/

2. ubuntu中启动nfs , tftp服务
<font color=red>设置nfs服务配置并启动：</font>
使用sudo vim /etc/exports打开文件并添加如下内容：
```c
/home/lsh/linuxdriver/nfs *(rw,sync,no_root_squash) 
`````

<可选>:这个步骤可以在最终挂在出错的情况下进行，解决nfs版本冲突问题（正点原子教程没有提及）

打开文件：sudo vim /etc/default/nfs-kernel-server
![1]() 

保存退出,使用sudo service nfs-kernel-server restart重启nfs服务

<font color=red>设置tftp配置并启动服务</font>

参考唐部长


3. ubuntu中将zImage , dtb , 根文件系统放入对应位置
最后是在rootfs里面放入文件系统内容，如下所示：
在tftpboot下放入如下内容

4. 启动
进入uboot后设置参数：参考上方uboot参数。最后输入boot命令启动
