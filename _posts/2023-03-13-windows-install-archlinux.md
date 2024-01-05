---
layout:       post
title:        "Windows11 wsl2 安装archlinux子系统"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - ArchLinux
---

这篇文章针对的是在win11系统的wsl2下安装ArchLinux系统，网上很多中文教程都是使用LxRunOffline去做的，但是实际上该方法已经过时了，目前有更加先进的ArchWSL方式。

如果用的是wsl1，不保证本教程可以适用。

## 安装ArchLinux子系统

首先，需要确保你的系统已经安装并打开wsl2功能。详见[官方文档](https://learn.microsoft.com/zh-cn/windows/wsl/install)。

Github上的[ArchWSL](https://github.com/yuk7/ArchWSL)项目已经帮我们把ArchLinux集成好了，可以到下载页面下载最新的Arch.zip文件：[下载页面](https://github.com/yuk7/ArchWSL/releases/latest)。

下载好之后，解压其中的文件到你需要存放ArchLinux的路径，例如D:\vm\archlinux。随后执行目录下的Arch.exe文件，安装程序会自动将ArchLinux安装到同目录下面，并配置好wsl。

安装完成之后，打开终端，应该可以看到刚装好的ArchLinux系统：

```powershell
wsl --list
```

可以看到我的电脑上除了ArchLinux之外还有别的子系统，你可以保留它们，也可以使用下面的命令卸载：

```powershell
wsl --unregister Ubuntu
```

如有需要，使用下面命令将ArchLinux设为默认系统：

```powershell
wsl --set-default Arch
```

使用下面的命令就可以进入ArchLinux了（如果你把ArchLinux设为默认系统了，则可以省略参数）：

```powershell
wsl -d Arch
```

进入系统之后，会做一些配置，结束之后就可以进入bash shell了。

## 配置pacman

首先，配置pacman镜像源，改为国内的。

```bash
vim /etc/pacman.d/mirrorlist
```

增加以下内容：

```toml
Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
```

输入下面命令，配置pacman key：

```bash
pacman-key --init
pacman-key --populate
pacman -S archlinux-keyring
```

更新系统：

```bash
pacman -Syu
```

配置archlinuxcn镜像源：

```bash
vim /etc/pacman.conf
```

增加以下内容：

```toml
[archlinuxcn]
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

安装archlinuxcn的key：

```bash
pacman -S archlinuxcn-keyring
```

## 配置用户

为root设置密码：

```bash
passwd
```

配置sudo：

```bash
echo "%wheel ALL=(ALL) ALL" > /etc/sudoers.d/wheel
```

添加非root用户：

```bash
useradd -m -G wheel -s /bin/bash {username}
passwd {username}
```

退出ArchLinux，进入刚刚安装ArchLinux的目录（例如D:\vm\archlinux），将默认用户改为非root用户：

```powershell
exit
cd D:\vm\archlinux
.\Arch.exe config --default-user {username}
```

重启wsl并再次进入ArchLinux，你应该会进入非root用户：

```powershell
wsl --shutdown  # 这个命令会关闭所有虚拟机
wsl -d Arch
```

## 完成

以上，你就得到了最小的ArchLinux系统，如果要进行进一步配置，需要参考官方文档。

如果涉及到wsl的操作，例如要安装x-server以支持图形界面，或是配置GPU直连，可以参考微软官方的wsl文档：[Windows Subsystem for Linux Documentation](https://learn.microsoft.com/en-us/windows/wsl/)

如果是ArchLinux本身的操作，请参考[wiki](https://wiki.archlinux.org/)，另外我个人推荐一篇很好的ArchLinux入门中文教程：[archlinux 简明指南](https://arch.icekylin.online/)。