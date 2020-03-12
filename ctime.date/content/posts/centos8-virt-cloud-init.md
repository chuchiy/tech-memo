---
title: "基于cloud init的CentOS 8虚拟化管理"
date: 2020-03-12T14:39:26+08:00
draft: true
---

## Cloud Init介绍
[cloud-init](https://cloudinit.readthedocs.io/en/latest/)是一种自动化的在云环境中配置初始化虚拟机的机制, 可以通过虚拟机挂载一个预先配置好的发行版Cloud磁盘镜像与配置文件ISO文件直接启动得到一个配置好的虚拟机，目前各个主流发行版都有官方支持的Cloud镜像.

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

在我自己的设备上，这个命令会返回一个Warning提示IOMMU未激活，需要将intel_iommu=on加到kernel启动参数中, 根据[Redhat内核启动参数配置文档], 应该可以通过修改/etc/default/grub
中的GRUB_CMDLINE_LINUX, 然后执行grub2-mkconfig -o /boot/grub2/grub.cfg来添加，但这种方法经我测试没有效果，查看其他资料有所到可以需要UEFI设置可能需要修改/boot/efi/中的配置，但测试也没有效果，在linux启动是直接修改grub启动参数加上intel_iommu=on是可以的，具体原因待查。考虑使用[grubby工具]来修改

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



## 准备cloud init文件

参考[Redhat 7的cloud init配置文档], 

https://stafwag.github.io/blog/blog/2019/03/03/howto-use-centos-cloud-images-with-cloud-init/

## 虚拟机管理


## 下一步工作
- intel_iommu如何在grub中正确配置
- #cloud-config中默认用户的sudo是否就是NOPASSWORD:ALL?
- 如何在meta-data中配置网络, 参考RHEL7和cloud-init官方文档
- CentOS 8 libvirt将虚拟机网络直接桥接到路由器上的配置

[Redhat虚拟化配置文档]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_virtualization/getting-started-with-virtualization-in-rhel-8_configuring-and-managing-virtualization#creating-vms-using-the-rhel-8-web-console_assembly_creating-virtual-machine
[Redhat内核参数配置文档]:https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/configuring-kernel-command-line-parameters_managing-monitoring-and-updating-the-kernel
[Redhat 7的cloud init配置文档]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/installation_and_configuration_guide/setting_up_cloud_init
[grubby工具]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/sec-making_persistent_changes_to_a_grub_2_menu_using_the_grubby_tool