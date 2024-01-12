---
layout:       post
title:        "ArchLinux 安装VMWare并模拟Windows 11"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - ArchLinux
---

在QQ已经推出基于electron的QQ For Linux之后，微信仍然不为所动。由于我日常工作开发都是在ArchLinux下完成的，而工作又需要用到企业微信这个软件。如何在ArchLinux下使用微信变成了困扰我很久的问题。

尝试过qq.weixin.work.deepin这种基于wine的方案，但是遇到过包括图形bug、作者不再维护等问题。

微信这种软件还是在Windows或MacOS下原生使用是最舒服的。而我的ArchLinux机器配置比较好，遂考虑安装VMWare模拟Windows系统专门用来运行企业微信。

首先，需要安装linux-headers这个包：

```bash
sudo pacman -S linux-headers
```

从AUR中一键安装VMWare：

```bash
yay -S vmware-workstation
```

这会自动安装并构建VMWare所需要的一些内核模块，因此你不需要再手动执行`vmware-mod-config`。

如果你在启动虚拟机时报错："could not open /dev/vmmon"，说明yay构建内核模块失败了，或者linux-headers没有安装。

手动构建VMWare所需要的内核模块：

```bash
sudo vmware-modconfig --console --install-all
```

如果执行报错，需要手动make(请注意确保你的linux-headers已经安装)：

```bash
git clone https://github.com/mkubecek/vmware-host-modules.git
cd vmware-host-modules
git checkout -b 16.2.1 origin/workstation-16.2.1
sudo make
```

如果在创建虚拟机之后，报错："failed to connect virtual device 'ethernet0'"，可能是VMWare网络组件没有启动，执行下面命令即可解决：

```bash
sudo systemctl enable --now vmware-networks
```

如果还是遇到问题，可以查看日志：`~/vmware/<your-vm-name>/vmware.log`。