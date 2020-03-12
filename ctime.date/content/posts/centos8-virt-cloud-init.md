---
title: "基于cloud init的CentOS 8虚拟化管理"
description: "使用cloud-init自动化配置虚拟机"
date: 2020-03-12T14:39:26+08:00
---

## Cloud Init介绍
[cloud-init](https://cloudinit.readthedocs.io/en/latest/)是各大云服务提供商广泛使用的自动化的在云环境中配置初始化虚拟机的机制, 在单机上应用时可以通过虚拟机挂载一个预先配置好的发行版Cloud磁盘镜像与配置文件ISO文件直接启动得到一个配置好的虚拟机，目前各个主流发行版都有官方支持的Cloud镜像.

本文以在CentOS 8宿主机中创建CentOS 8虚拟机为例介绍

## 配置检查虚拟化

基本流程参考[Redhat虚拟化配置文档]，安装虚拟化工具

```
yum module install virt
yum install virt-install virt-viewer
```

检查宿主机配置

```
virt-host-validate
```

在我自己的设备上，这个命令会返回一个Warning提示IOMMU未激活，需要将intel_iommu=on加到kernel启动参数中, 根据[Redhat内核启动参数配置文档], 应该可以通过修改`/etc/default/grub`
中的`GRUB_CMDLINE_LINUX`, 然后执行`grub2-mkconfig -o /boot/grub2/grub.cfg`来添加。

但这种方法经我测试没有效果，查看其他资料有所到可以需要UEFI设置可能需要修改/boot/efi/中的配置，但测试也没有效果。

在linux启动是直接修改grub启动参数加上intel_iommu=on是可以的，具体原因待查。考虑使用[grubby工具]来修改

## 准备cloud版本镜像

获取CentOS 8的Cloud镜像:

访问https://cloud.centos.org/centos/8/x86_64/images/ 下载最新版本的CentOS-8-GenericCloud-******.x86_64.qcow2，
下完后检查看一下这个镜像:

```
$ qemu-img info CentOS-8-GenericCloud-8.1.1911-20200113.3.x86_64.qcow2 
image: CentOS-8-GenericCloud-8.1.1911-20200113.3.x86_64.qcow2
file format: qcow2
virtual size: 10G (10737418240 bytes)
disk size: 683M
cluster_size: 65536
Format specific information:
    compat: 0.10
    refcount bits: 16
```

镜像默认磁盘是10G，一般都需要放大，复制原始文件到存放虚拟机磁盘文件的目录，将文件大小更改为需要的尺寸

```
cp -v CentOS-8-GenericCloud-8.1.1911-20200113.3.x86_64.qcow2 /data/vm/centos8-test.qcow2
qemu-img resize /data/vm/centos8-test.qcow2 20G
```

如果系统的selinux启用了，需要调整虚拟机磁盘文件的属性，添加virt_image_t类型

```
sudo semanage fcontext -a -t virt_image_t '/data/vm/centos8-test.qcow2'
restorecon -v '/data/vm/centos8-test.qcow2'
```

## 准备cloud init文件

参考[Redhat 7的cloud init配置文档], 准备meta-data和user-data两个文件

meta-data中定义虚拟机的主机名之类信息，如果有格外的网络配置也在里面(默认是DHCP获取IP)

```
instance-id: Centos8test
local-hostname: centos8-test
```

user-data中可以配置账号等信息

```
#cloud-config
user: testuser
password: testpass
groups: wheel
chpasswd: {expire: False}
ssh_authorized_keys:
  - you-public-key
```

在yaml文件根上配置的用户信息是覆盖的cloud-init默认的cloud-user用户的信息，这个用户应该默认会配置无密码sudo的权限，你在这里不配置sudo也会有sudo权限

user-data的详细配置还可以见[cloud-init的对应文档](https://cloudinit.readthedocs.io/en/latest/topics/examples.html)

用户密码除了直接写明文以外也可以写hash, 可以用下面这段代码直接执行输入密码后得到密码hash值

```
python3 -c 'import crypt,getpass; print(crypt.crypt(getpass.getpass(), crypt.mksalt(crypt.METHOD_SHA512)))'
```

user-data和meta-data文件都准备好之后需要制作一个ISO在虚拟机启动的时候挂载

```
genisoimage -output /data/vm/centos8-test-cidata.iso -volid cidata -joliet -rock user-data meta-data
```

## 虚拟机管理

使用virt-install初始化新的虚拟机, 使用默认的dhcp/nat网络

```
sudo virt-install --name centos8-test1 --memory 1024 --vcpus 1 --os-variant rhel8.0 --import --disk /data/vm/centos8-test.qcow2 --disk /data/vm/centos8-test-cidata.iso --graphics none
```

使用virsh来管理虚拟机，注意virsh需要以root权限执行，否则virsh看不到在运行的虚拟机

```
virsh # list
 Id    Name                           State
----------------------------------------------------
 7     centos8-test1                  running

virsh # net-dhcp-leases default
 Expiry Time          MAC address        Protocol  IP address                Hostname        Client ID or DUID
-------------------------------------------------------------------------------------------------------------------
 2020-03-12 18:09:05  52:54:00:3a:ab:a0  ipv4      192.168.122.65/24         centos8-test    01:52:54:00:3a:ab:a0
```

停止虚拟机为virsh destroy ...

删除虚拟机定义(磁盘还在)为virsh undefine ...



## 下一步工作
- intel_iommu如何在grub中正确配置
- #cloud-config中默认用户的sudo是否就是NOPASSWORD:ALL?
- 如何在meta-data中配置网络, 参考RHEL7和cloud-init官方文档
- CentOS 8 libvirt将虚拟机网络直接桥接到路由器上的配置

[Redhat虚拟化配置文档]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_virtualization/getting-started-with-virtualization-in-rhel-8_configuring-and-managing-virtualization#creating-vms-using-the-rhel-8-web-console_assembly_creating-virtual-machine
[Redhat内核启动参数配置文档]:https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/configuring-kernel-command-line-parameters_managing-monitoring-and-updating-the-kernel
[Redhat 7的cloud init配置文档]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/installation_and_configuration_guide/setting_up_cloud_init
[grubby工具]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/sec-making_persistent_changes_to_a_grub_2_menu_using_the_grubby_tool