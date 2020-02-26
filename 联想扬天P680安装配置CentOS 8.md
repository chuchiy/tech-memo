# 联想扬天P680安装配置CentOS 8

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

## 安装Linux目标
- 使用CentOS 8, 在设备生命周期类操作系统不更换
- 桌面CPU的个人工作站, 显示驱动正确
- 运行CUDA的设备
- 运行Podman的容器化宿主

## 问题解决

### 安装程序图形界面启动
默认情况下显示从NV独立显卡输出，Debian 10/Ubuntu 19.04从U盘安装引导后都无法显示图形安装界面，CentOS 8在无法显示图形安装界面后会Fallback到文本安装界面，原因应该是无法正确驱动NV显卡。需要在BIOS中配置使用CPU自带的核显，将显示输出接到核显的HDMI上，这样可以正常安装，NV显卡在安装结束后再配置(见后)

### 无线网卡驱动
这个机器的无线网卡型号rtl8821ce, linux内核和各个发行版中都没有驱动，目前有一个民间维护的驱动<https://github.com/tomaspinho/rtl8821ce>。这个驱动是为Arch Linux和Ubuntu设计的，标准安装流程：

```
git clone https://github.com/tomaspinho/rtl8821ce
sudo dnf install epel-release -y
sudo dnf install kernel-devel dkms -y
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