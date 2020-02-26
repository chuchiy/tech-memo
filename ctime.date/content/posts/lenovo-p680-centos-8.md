---
title: "联想扬天P680安装配置CentOS 8"
date: 2020-02-26T16:30:17+08:00
description: "为CentOS 8安装rtl8821ce无线网卡驱动"
tags: ["技巧"]
---

## 机器配置

- i5-9400
- 8G DDR4 2666 x 1 (SK Hynix) 
- GTX 1660Ti 6GB
- 256G NVME SSD (Samsung)
- 1TB 硬盘
- 有线/无线网卡
- 带一个3.5寸热插拔硬盘槽
- 带Win10
- 长32.6 宽16.5 高40.4 18L机箱

自己加了一根Apacer Panther 8GB DDR4 2666内存，使用没有问题，但内存会降频到2133

## 安装Linux

### 使用目标
- 使用CentOS 8, 在设备生命周期类操作系统不更换
- 桌面CPU的个人工作站, 显示驱动正确
- 运行CUDA的设备
- 运行Podman的容器化宿主

### 安装流程
安装linux/windows双系统，在windows中在SSD中用压缩硬盘为linux留出足够的未分配空间(如100G)，U盘启动CentOS 8安装，在分区的地方选择SSD，使用手动分区模式，让程序自动分配存储卷和挂载点，安装程序应该可以自动将未分配磁盘空间都使用掉并保留windows的分区，安装角色使用workstion，CentOS会自动配置好双启动。

## 问题解决

### 安装程序图形界面启动
默认情况下显示从NV独立显卡输出，Debian 10/Ubuntu 19.04从U盘安装引导后都无法显示图形安装界面，CentOS 8在无法显示图形安装界面后会Fallback到文本安装界面，原因应该是无法正确驱动NV显卡。需要在BIOS中配置使用CPU自带的核显，将显示输出接到核显的HDMI上，这样可以正常安装，NV显卡在安装结束后再配置(见后)

### 无线网卡驱动
这个机器的无线网卡型号rtl8821ce, linux内核和各个发行版中都没有驱动，目前有一个民间维护的驱动<https://github.com/tomaspinho/rtl8821ce>。这个驱动是为Arch Linux和Ubuntu设计的，标准安装流程：

```
sudo dnf install epel-release -y
sudo dnf install kernel-devel dkms -y
sudo dnf group install "Development Tools"
git clone https://github.com/tomaspinho/rtl8821ce
cd rtl8821ce
sudo sh ./dkms-install.sh
```

这里编译会出错，需要修改2个地方。

对access_ok系统调用的使用: 标准linux内核中，5.0以前access_ok接收3个参数，5.0后access_ok接收2个参数。尽管CentOS 8内核版本为4.18，access_ok调用应该是从5.0+的内核backport回来的接收2个参数，需要调整`os_dep/linux/rtw_android.c`中的代码

```c
#if (LINUX_VERSION_CODE >= KERNEL_VERSION(5,0,0))
	if (!access_ok(priv_cmd.buf, priv_cmd.total_len)) {
#else
	if (!access_ok(VERIFY_READ, priv_cmd.buf, priv_cmd.total_len)) {
#endif
```

将检查的版本临时改为CentOS 8的内核版4.18，这个修改会破坏代码在标准的linux内核上的编译，如何正确的Patch还需要研究

编译Warning处理：代码中有一个地方会有incompatible-pointer-types的警告导致编译失败，需要修改Makefile
在开始的地方加一个`EXTRA_CFLAGS += -Wno-incompatible-pointer-types`跳过这类错误

处理了这两个问题后可正常通过DKMS安装编译，无线网卡使用没有问题
   

### NV显卡驱动与X11桌面

设备配置使用集成显卡安装结束后进入Linux桌面会有两个问题:

1. 集显的HDMI版本为1.4，4K输出刷新率只能是30Hz
2. 在集显模式下操作系统是看不到NV的独立显卡的

因此需要先安装NV官方的[CUDA 10.2](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&target_distro=CentOS&target_version=8&target_type=rpmlocal)，安装结束后执行:

```
nvidia-xconfig #生成X11桌面的配置
systemctl set-default multi-user.target #将系统的runlevel设成命令行，原因下述
```

重启进入BIOS，将显卡使用更改为使用独立显卡，将视频输出接到独立显卡上，系统启动后会先进入命令行登录，进入命令行后执行`systemctl isolate graphical.target`进入图形登录界面，输入用户密码进入用户桌面。

目前的问题是如果我直接将系统的runlevel设置为graphical.target，系统启动后可以进入图形登录界面，但在输入用户名密码后是无法进入桌面的，一直黑屏，查相关日志能看到X11报错，具体的解决方法还需要进一步研究。

### 硬件时钟不是UTC
Windows在默认情况下认为硬件时钟是本地时间，Linux默认认为硬件时钟是UTC时间，Linux需要配置

```
sudo timedatectl set-timezone Asia/Shanghai
sudo systemctl enable chronyd
sudo systemctl start chronyd
sudo hwclock --systohc --localtime
```


## 下一步工作
* 正确的Patch目前的rtl8821ce驱动，让其默认编译可以在Centos 8以及其他发行版上可以工作
* 研究开机启动直接进入图形模式登录后无法进入用户桌面的问题
