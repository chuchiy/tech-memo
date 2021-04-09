## ubuntu live 配置

### 启动配置调整


打开启动U盘的`/boot/grub/grub.cfg`, 修改 

```bash
linux	/casper/vmlinuz  persistent file=/cdrom/preseed/ubuntu.seed quiet splash ..... ---
```

- 添加`fsck.mode=skip` 禁用启动时的磁盘检查
- 添加`noprompt` 禁用关机重启时弹出U盘的提示, 参见[capser manpage](http://manpages.ubuntu.com/manpages/bionic/man7/casper.7.html)

修改后如:

```bash
linux	/casper/vmlinuz  persistent file=/cdrom/preseed/ubuntu.seed fsck.mode=skip noprompt quiet splash ---
```

