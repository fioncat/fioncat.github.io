---
layout:       post
title:        "ArchLinux简明安装指南"
subtitle:     "Install archlinux guide"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - ArchLinux
    - Linux
---

![final](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/final.png)

ArchLinux是很少见的连安装都让我受益匪浅的操作系统，她作为我的开发机主力系统，已经从很多层面上赢得了我的青睐。

ArchLinux的安装经常劝退了不少初学者，事实上这个过程能让你对你自己的系统有更加全面的理解。但是即使是我这种ArchLinux<rm>老手<rm/>，在一段时间之后，也会忘记ArchLinux的一些安装步骤。所以记录了这个文档，以快速复盘安装过程。

本文的内容主要来自于一个非常优秀的简体中文ArchLinux安装教程：[archlinux 简明指南](https://arch.icekylin.online/)。在其基础上，基于我自己的偏好做了删改，主要是：

- 移除Windows双系统的相关内容。双系统会带来一些潜在的坑，建议将Windows装在不同磁盘上面。
- 移除虚拟机的相关内容，实体机中体验ArchLinux才是最好的！
- 由于我是用ArchLinux作为主力开发机的。移除了一些我用不到的软件，增加更多适合后端开发者的内容。

本文限定你的主板是支持UEFI的，如果用的还是只支持BIOS的上古设备，请查阅别的资料。

ArchLinux的一个顶级优势是她那无比完备的[ArchWiki](https://wiki.archlinux.org/)，在安装中遇到的任何问题都可以到wiki上面搜索对应的解决方案。

## 准备

跟别的发行版不同，最好在每次安装ArchLinux之前，都重新制作一个新的启动U盘。因为官方会在每个月放出一个新的iso镜像文件，**如果用过旧的镜像来安装ArchLinux，很可能会失败（因为签名等原因）。**

可以到[archlinux download](https://archlinux.org/download/)页面选择不同的镜像站下载最新的iso文件。

下载完成之后，需要制作启动U盘，在Linux下面可以使用`dd`命令来制作：

> ⚠️警告：这会导致你U盘的所有数据丢失

```bash
lsblk   # 列出设备，记住你的启动盘设备名，假设为/dev/sdx
sudo dd bs=4M if=/path/to/archlinux.iso of=/dev/sdx status=progress oflag=sync
```

如果你用的是MacOS系统，同样可以用`dd`命令来制作，但是在制作前需要对iso文件进行格式转换：

```bash
hdiutil convert -format UDRW -o archlinux.dmg archlinux.iso
```

完成启动盘制作：

```bash
diskutil list # 列出设备，找到你的U盘（可以通过大小来区分）
diskutil unmountDisk /dev/diskx # 卸载U盘
sudo dd bs=4m if=archlinux.dmg of=/dev/disk4 status=progress
```

关于Windows如何制作启动盘，可能需要借助第三方软件来完成，这里不再赘述，有需要请自行参考网上资料。

接下来，插入启动盘，通过U盘引导，选择第一个选项：

![archiso-01](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/archiso-01.png)

经过加载之后，你可以看到下面提示符：

![archiso-02](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/archiso-02.png)

这说明你已经进入了由U盘引导的`archiso`，这是一个最小化的arch系统，用于系统安装和维护。

强烈建议常备一个ArchLinux启动盘，万一某一天滚挂了，你会发现连系统都进入不了（指的grub引导挂了），这时候可以通过`archiso`的`arch-chroot`临时进入到系统中，以进行紧急修复，包括重建grub或回滚磁盘快照等操作。详见后面关于系统恢复的章节。尽管现在ArchLinux滚挂的情况已经比较少了（从我多年使用到今从来没有出现过），但是常言道，有备无患嘛。

## 基础安装

基础安装将完成不包括桌面环境的安装。

### UEFI

确认我们当前处于UEFI模式：

```bash
ls /sys/firmware/efi/efivars  # 此命令会列出UEFI变量
```

### 禁用reflector

首先，禁用reflector服务，这个服务会覆盖我们的mirrorlist，对于国内环境来说并不友好：

```bash
systemctl stop reflector.service
systemctl status reflector.service
```

### 连接网络

ArchLinux的安装必须依赖网络，如果用的是有线连接，此时应该自动连接上了网络。如果需要用无线网络，则需要使用`iwctl`命令。

连接网络之后，需要通过网络更新系统时钟：

```bash
timedatectl set-ntp true # 将系统时间与网络时间进行同步
timedatectl status # 检查服务状态
```

### 更换国内镜像源

将`pacman`镜像源换成国内的可以大大提高安装速度：

```bash
vim /etc/pacman.d/mirrorlist
```

修改为：

```text
Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
```

### 初始化磁盘GPT

> ⚠ 警告：下面的步骤涉及对磁盘的操作，每一步都可能会导致数据丢失，请务必谨慎操作！在按下回车执行命令之前一定要确认自己在做什么！系统可以重新装，数据却是无价的。

如果你手中的磁盘还没有任何系统，则需要先建立GPT分区表。我们先确认要安装ArchLinux的磁盘设备号：

```bash
lsblk  # 确认磁盘设备号，假设为nvme0n1
```

这里我们假设要安装系统的设备号为`nvme0n1`，如果你用的是基于NVMe的SSD磁盘，一般就是这个磁盘号。如果你用的是传统的SATA磁盘，磁盘号则一般为`sdx`。你可以根据`lsblk`命令输出的磁盘大小来确定要安装ArchLinux到哪个磁盘中。

如果你的盘号并不是`nvme0n1`，请在下面执行分区等命令时将盘号替换为你自己的。

将磁盘转换为gpt类型（再次警告，这里会抹除磁盘的所有数据）：

```bash
parted /dev/nvme0n1  # 进行磁盘类型变更，会进入一个命令交互
(parted) mktable # 更改磁盘类型
New disk label type? gpt  # 输入gpt，将磁盘变更为gpt类型
Warning: xxx # 如果磁盘已经有数据，这里会打印警告
Yes/No? Yes # 输入Yes确认抹除所有数据
(parted) quit # 退出命令交互
```

### 磁盘分区

我们将建立3个分区：

- nvme0n1p1：EFI分区，用于引导系统启动。
- nvme0n1p2：swap分区，用于缓存内存数据。
- nvme0n1p3：主分区，将采用btrfs，并用子卷继续对该分区进行划分。

通过下面的命令开始分区：

```bash
cfdisk /dev/nvme0n1
```

这会进入一个`tui`，里面会列出所有现存的分区。我们可以通过方向上下选择要操作的分区，左右选择要进行的操作。

选中下面的`Free space`，创建新的分区。注意创建的分区默认类型为`Linux filesystem`，我们需要手动修改类型（选择下方的Type操作即可更改类型）：

- nvme0n1p1：EFI分区大小建议为512MB，类型为`EFI System`。
- nvme0n1p2：Swap分区大小建议为内存的`60%`，类型为`Linux swap`。
- nvme0n1p3：主分区大小可以根据实际情况分配，默认为剩下的所有空间，类型为`Linux filesystem`（默认，无需修改）。

创建完成之后，选中操作`Write`以将分区写入，随后退出`cfdisk`工具。

完成之后，通过下面的命令可以复盘分区情况：

```bash
fdisk -l
```

### 格式化分区

分区创建之后，需要进行格式化才能使用。

格式化EFI分区：

```bash
mkfs.vfat /dev/nvme0n1p1
```

格式化Swap分区：

```bash
mkswap /dev/nvme0n1p2
```

格式化btrfs分区：

```bash
mkfs.btrfs -L arch /dev/nvme0n1p3
```

> -L参数表示分区Label，可以自定义

### 创建btrfs逻辑子卷

我们打算为根目录`/`和家目录`/home`创建不同的逻辑子卷。这也是常见的做法。

为了创建子卷，先将btrfs挂载到`/mnt`下面：

```bash
mount -t btrfs -o compress=zstd /dev/nvme0n1p3 /mnt
df -h  # 确认磁盘的挂载情况
```

创建两个子卷：

```bash
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume list -p /mnt  # 列出刚创建的子卷
```

> ⚠ 警告：快照工具`timeshift`只支持这种子卷布局，如果你采用不同的布局，`timeshift`可能会存在问题。

完成之后，卸载btrfs：

```bash
umount /mnt
```

### 挂载磁盘

我们已经完成了新系统磁盘的格式化，下面需要将磁盘挂载到最小系统上面，以准备进行ArchLinux的正式安装。

首先，挂载btrfs子卷：

```bash
mount -t btrfs -o subvol=/@,compress=zstd /dev/nvme0n1p3 /mnt
mkdir /mnt/home
mount -t btrfs -o subvol=/@home,compress=zstd /dev/nvme0n1p3 /mnt/home
```

挂载EFI：

```bash
mkdir -p /mnt/boot/efi
mount /dev/nvme0n1p1 /mnt/boot/efi
```

挂载交换分区：

```bash
swapon /dev/nvme0n1p2
```

使用下面命令查看磁盘挂载情况：

```bash
df -h
```

使用下面命令查看交换分区挂载情况：

```bash
free -h
```

### 安装系统

终于，万事俱备，只欠东风，我们可以开始安装ArchLinux了。

可以通过官方工具`pacstrap`直接完成安装：

```bash
pacstrap /mnt base base-devel linux linux-firmware
```

安装其他必要的功能性软件：

```bash
pacstrap /mnt dhcpcd iwd vim sudo zsh zsh-completions
```

### 生成fstab文件

fstab文件定义了磁盘分区，需要被Linux识别。可以使用下面的命令生成：

```bash
genfstab -U /mnt > /mnt/etc/fstab
cat /mnt/etc/fstab  # 确保生成正确
```

为了让`timeshift`能正常恢复btrfs，需要修改`/mnt/etc/fstab`，将其中的`subvolid=xxx`去掉：

```bash
# 编辑文件，去除所有的subvolid=xxx挂载选项
vim /mnt/etc/fstab
cat /mnt/etc/fstab | grep subvolid # 确认已经没有subvolid了
```

### 切换到新系统

现在，新系统已经准备好了，我们可以使用chroot切换到新系统下面：

```bash
arch-chroot /mnt
```

## 基础配置

我们现在得到了一个没有图形化界面的新ArchLinux系统，下面进行一些基本的配置。

### 设置主机名和时区

为新的系统设置一个主机名吧：

```bash
vim /etc/hostname
```

假设主机名为`myarch`，在`hosts`文件加入对应内容：

```bash
vim /etc/hosts
```

加入：

```text
127.0.0.1   localhost
::1         localhost
127.0.0.1   myarch.localdomain	myarch
```

设置时区为上海：

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

将系统时间同步到硬件：

```bash
hwclock --systohc
```

### 语言设置

设置locale：

```bash
vim /etc/locale.gen
```

在其中找到`en_US.UTF-8 UTF-8`和`zh_CN.UTF-8 UTF-8`，去掉前面的注释即可。

生成locale：

```bash
locale-gen
```

设置当前语言为英文（`tty`下不建议设置为中文）：

```bash
echo 'LANG=en_US.UTF-8'  > /etc/locale.conf
```

### 设置root密码

执行下面的命令为root用户设置密码：

```bash
passwd root
```

### 安装微码

根据你的CPU类型安装不同制造商的微码：

```bash
pacman -S intel-ucode # Intel
pacman -S amd-ucode # AMD
```

### 安装引导程序

我们将会使用`grub`来引导ArchLinux，安装：

```bash
pacman -S grub efibootmgr os-prober
```

将grub安装到EFI分区：

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ARCH
```

接下来，编辑grub配置文件：

```bash
vim /etc/default/grub
```

建议将`GRUB_CMDLINE_LINUX_DEFAULT`修改为`loglevel=5 nowatchdog`。其他参数可以根据自己的喜好进行修改。

生成grub所需要的配置文件：

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

完成之后，就可以用grub来引导新系统了，我们直接重启（注意重启后要拔掉启动盘，不然还可能会进入archiso环境）：

```bash
exit # 退出chroot环境
reboot
```

重启之后，如果正常进入新的archlinux系统，会要求你登录，直接用root用户登录即可。

进入新系统后第一件事是检查能否上网，如果不能，可能需要启动`dhcp`服务：

```bash
systemctl enable --now dhcpcd
```

如果用的wifi，需要启动`iwd`服务：

```bash
systemctl start iwd # 立即启动 iwd
iwctl # 和之前的方式一样，连接无线网络
```

### 创建非root用户

我们不能在root用户上面进行日常操作，需要创建一个担当管理员角色的普通用户。

创建用户并设置密码：

```bash
useradd -m -G wheel -s /bin/bash <user-name>
passwd <user-name>
```

编辑`sudoers`文件，给这个用户加上sudo权限：

```bash
EDITOR=vim visudo
```

找到下面这行，将注释去掉：

```text
%wheel ALL=(ALL) ALL
```

完成之后，`logout`注销root，就能用新的非root用户登录系统了。

### pacman配置

编辑pacman配置文件：

```bash
sudo vim /etc/pacman.conf
```

去掉`multilib`相关注释，以开启32位库支持。

在配置最后加上下面内容，以添加`archlinuxcn`镜像源：

```toml
[archlinuxcn]
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

退出，更新pacman：

```bash
sudo pacman -Syyu
```

## 安装桌面环境

没有图形界面的tty系统并不适用于日常使用。下面我们来对archlinux安装[KDE Plasma](https://kde.org/plasma-desktop/)桌面环境。ArchLinux可以选择其它桌面环境，有需求可以自行参考文档。

安装kde：

```bash
sudo pacman -S plasma-meta konsole dolphin
```

开启sddm守护进程：

```bash
sudo systemctl enable sddm
```

启动桌面，输入后应该可以看到欢迎界面：

```bash
sudo systemctl start sddm
```

只要不做过多的私人定制，KDE本身还是比较稳定的。

## 桌面基础配置

输入密码进入桌面之后，可以打开`Konsole`终端，继续进行我们的安装和配置。

### 基础软件安装

新的桌面环境是没有网络管理功能的，我们需要手动开启：

```bash
sudo systemctl disable iwd # 确保 iwd 开机处于关闭状态，因为其无线连接会与 NetworkManager 冲突
sudo systemctl stop iwd # 立即关闭 iwd
sudo systemctl enable --now NetworkManager # 确保先启动 NetworkManager，并进行网络连接。若 iwd 已经与 NetworkManager 冲突，则执行完上一步重启一下电脑即可
ping www.bilibili.com # 测试网络连通性
```

安装一些基础软件：

```bash
sudo pacman -S adobe-source-han-sans-cn-fonts adobe-source-han-serif-cn-fonts # 中文字体，推荐思源字体
sudo pacman -S noto-fonts noto-fonts-cjk noto-fonts-emoji noto-fonts-extra # 安装谷歌字体以及emoji支持
sudo pacman -S nerd-fonts-complete # nerd字体
sudo pacman -S ntfs-3g # 使系统可以识别 NTFS 格式的硬盘
sudo pacman -S ark # 压缩软件。在 dolphin 中可用右键解压压缩包
sudo pacman -S packagekit-qt5 packagekit appstream-qt appstream # 确保 Discover（软件中心）可用，需重启
sudo pacman -S gwenview # 图片查看器
```

`yay`，用于安装[AUR](https://aur.archlinux.org/packages)软件，需要从archlinuxcn进行安装：

```bash
sudo pacman -S archlinuxcn-keyring
sudo pacman -S yay  # 来自archlinuxcn源
```

注意，如果安装`archlinuxcn-keyring`报错：

```text
error: archlinuxcn-keyring: Signature from "Jiachen YANG (Arch Linux Packager Signing Key) <farseerfc@archlinux.org>" is marginal trust
```

执行下面命令即可解决：

```bash
sudo pacman-key --lsign-key "farseerfc@archlinux.org"
```

下面，就可以用`yay`来安装AUR软件了，例如chrome浏览器：

```bash
yay -S google-chrome
```

### 检查家目录

检查家目录下的各个常见目录是否已经创建，若没有则需通过以下命令手动创建：

```bash
mkdir Desktop Documents Downloads Music Pictures Videos
```

### 安装输入法

通过以下命令安装Fcitx5：

```bash
sudo pacman -S fcitx5-im # 输入法基础包组
sudo pacman -S fcitx5-chinese-addons # 官方中文输入引擎
sudo pacman -S fcitx5-material-color # 输入法主题
```

此外，我们还需要设置环境变量。通过 `vim` 编辑文件 `/etc/environment`：

```bash
sudo vim /etc/environment
```

在文件中加入以下内容并保存退出：

```environment
INPUT_METHOD=fcitx5
GTK_IM_MODULE=fcitx5
QT_IM_MODULE=fcitx5
XMODIFIERS=\@im=fcitx5
SDL_IM_MODULE=fcitx
```

重启或注销一下，应该就能输入中文了，使用`Ctrl + Space`快捷键可以切换输入法。

### 蓝牙

通过以下命令开启蓝牙相关服务并设置开机自动启动：

```bash
sudo systemctl enable --now bluetooth
```

如果无法连接蓝牙设备，安装下面的软件：

```bash
sudo pacman -S bluez bluez-utils pulseaudio-bluetooth
```

### 音频

如果系统没有声音，安装下面的软件：

```bash
sudo pacman -S pipewire-pulse pipewire-alsa pipewire-jack
```

或者尝试安装：

```bash
sudo pacman -S alsa-utils pulseaudio pavucontrol
```

### KDE桌面设置

修改单击文件的行为，选中而不是进入，更符合一般操作思维：

![desktop-setting](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/desktop-setting-click.png)

禁用桌面左上角边缘Action：

![desktop-setting](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/desktop-setting-edges.png)

## 透明代理

因为一些众所周知的原因，需要配置魔法上网才能正常使用系统。因为clash作者已经删库，因此这里推荐使用`v2ray`+`v2rayA`实现透明代理。

安装：

```bash
sudo pacman -S v2ray v2raya
```

安装后启动服务：

```bash
sudo systemctl enable --now v2raya
```

在浏览器中打开`127.0.0.1:2017`，或是搜索打开`v2rayA`应用，即可进入代理配置页面。

一开始会让你配置用户名和密码，请牢记。

进入配置页面后，点击右上角的`Import`将梯子供应商提供的订阅链接导入。

导入后，就可以选择对应的代理节点了：

![proxy-select](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/proxy-select.png)

随后，点击右上角的`Setting`，推荐进行以下设置：

![proxy-setting](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/proxy-setting.png)

- 代理策略选择不代理CN的站点。
- 代理模式建议选择`tproxy`，不然`NetworkManager`可能会报错。
- 建议开启防DNS污染，以解决某些域名（例如`raw.githubusercontent`）无法被解析的问题。

设置完成之后，就可以在左上角开启代理了：

![proxy-start](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/proxy-start.png)

之后系统重启代理也会默认生效，不需要重复开启了。

注意，开启代理之后可能出现无法进行`git clone`的问题，这时需要让ssh也支持走SSL协议，编辑`~/.ssh/config`，增加下面内容：

```
Host github.com
    Hostname ssh.github.com
    Port 443
    User git
```

## 编程软件

### 修改默认shell

我们可以将默认shell换成zsh：

```bash
chsh -l # 查看安装了哪些 Shell
chsh -s /usr/bin/zsh # 修改当前账户的默认 Shell
```

注销并重新登录，打开终端，会发现已经使用zsh作为默认shell。

### Alacritty

我们将不使用Konsole作为终端软件，因为它并不跨平台。除了Linux笔者还有一套Mac环境，因此我选择的[Alacritty](https://github.com/alacritty/alacritty)，这是一款用Rust编写的默认Terminal的替代品，可以利用GPU进行加速，性能强劲。但是功能比较简陋，没有标签页等功能，一般需要配合tmux来使用。

安装alacritty：

```bash
sudo pacman -S alacritty
```

新版本的Alacritty已经使用`toml`进行配置，导入配置文件：

```bash
mkdir -p ~/.config/alacritty
# 导入主题配色
curl https://raw.githubusercontent.com/fioncat/dotfiles/master/alacritty/catppuccin.toml > ~/.config/alacritty/catppuccin.toml
# 导入配置文件
curl https://raw.githubusercontent.com/fioncat/dotfiles/master/alacritty/alacritty.toml > ~/.config/alacritty/alacritty.toml
```

### oh-my-zsh

安装oh-my-zsh：

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

安装插件：

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

修改`.zshrc`，启用插件：

```bash
plugins=(
	zsh-autosuggestions
	zsh-syntax-highlighting

	vi-mode
	...
)
```

### fzf

必备的，先进的模糊搜索工具，安装：

```bash
sudo pacman -S fzf
```

我使用了[catppuccin](https://github.com/catppuccin/fzf)主题，将下面的语句加到`.zshrc`：

```bash
export FZF_DEFAULT_OPTS=" \
--color=bg+:#313244,bg:#1e1e2e,spinner:#f5e0dc,hl:#f38ba8 \
--color=fg:#cdd6f4,header:#f38ba8,info:#cba6f7,pointer:#f5e0dc \
--color=marker:#f5e0dc,fg+:#cdd6f4,prompt:#cba6f7,hl+:#f38ba8"
```

### 命令提示符

推荐使用[starship](https://github.com/starship/starship)作为默认命令行提示符，这是用Rust写的，功能和性能都非常强大：

```bash
sudo pacman -S starship
```

在`.zshrc`中添加下面内容以设为默认命令提示符：

```bash
eval "$(starship init zsh)"
```

注意，请使用nerd字体，另外确保在上面的步骤中已经安装了emoji支持，否则提示符会出现乱码。

### tmux

因为缺少标签页等功能，所以alacritty需要配合tmux才能使用，安装：

```bash
sudo pacman -S tmux
```

导入tmux配置：

```bash
curl https://raw.githubusercontent.com/fioncat/dotfiles/master/tmux/.tmux.conf > ~/.tmux.conf
curl https://raw.githubusercontent.com/fioncat/dotfiles/master/tmux/.tmux.conf.local > ~/.tmux.conf.local
```

这是我自己配的一个好看的tmux主题，可以配合catppuccin主题终端使用，效果：

![tmux](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/tmux.png)

### Neovim

[neovim](https://github.com/neovim/neovim)是我目前最喜欢的编辑器，它使用lua进行配置，可以做到终端下面的VSCode体验。

强烈推荐ayamir大大的nvim配置：[ayamir/nvimdots](https://github.com/ayamir/nvimdots)，安装方法见[wiki](https://github.com/ayamir/nvimdots/wiki/Prerequisites)。

我个人基于他的配置做了一些定制[fioncat/spacenvim](https://github.com/fioncat/spacenvim)：

- 固定插件版本，以防止某些插件更新导致的配置不兼容。提供rollback命令随时回退。
- 使用Space作为leader，加了一些特殊的emacs快捷键，保留我曾经使用[spacemacs](https://github.com/syl20bnr/spacemacs)的习惯。
- 插件列表更加轻量（移除了一些前端、C语言等插件），为Go和Rust开发者量身打造。

安装neovim：

```bash
sudo pacman -S neovim
```

一些nvim插件需要额外依赖：

```bash
sudo pacman -S sqlite3 fzf ripgrep xclip zip unzip npm nodejs
```

随后，一键安装spacenvim：

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/fioncat/spacenvim/HEAD/scripts/install.sh)"
```

安装之后，随意打开一个文件，让[treesitter](https://github.com/nvim-treesitter/nvim-treesitter)安装必要的parser。另外可以用`Mason`命令来安装各种语言的LSP支持。

现在，你可以跟`vim`说再见了：

```bash
alias vim="nvim"
```

效果：

![spacenvim](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/spacenvim.png)

### 命令行工具

Linux下很多命令都可以替换为更加现代的命令，例如ls，du，df，ps等。[morden-unix](https://github.com/ibraheemdev/modern-unix)这个仓库列出了常见的可以替换原始Linux命令的工具。

我们可以选择性进行安装，例如：

```bash
sudo pacman -S bottom duf exa dust procs
```

### 其他软件

推荐一些工作中经常用到的开发者软件。

**Code OSS：VSCode的开源构建，注意VSCode本身是闭源软件**

```bash
sudo pacman -S code
```

如果要安装vscode本体，执行：

```bash
yay -S visual-studio-code-bin
```

我的vscode配置：

```json
{
    "editor.fontSize": 16,
    "editor.fontFamily": "'SauceCodePro Nerd Font'",
    "editor.cursorBlinking": "solid",
    "editor.lineNumbers": "relative",
    "workbench.colorTheme": "Catppuccin Mocha",
    "debug.console.fontSize": 16,
    "chat.editor.fontSize": 16,
    "markdown.preview.fontSize": 16,
    "window.zoomLevel": 1,
    "editor.fontLigatures": false
}
```

推荐安装的插件：

- [vim](https://open-vsx.org/extension/vscodevim/vim)：在vscode下使用vim。
- [catppuccin](https://open-vsx.org/extension/Catppuccin/catppuccin-vsc)：我用的主题配色

效果：

![vscode](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/vscode.png)

**[Typora](https://typora.io/)：所见即所得的markdown编辑器，最新版收费。可以使用0.x的免费版**

```bash
yay -S typora-free  # 免费版
yay -S typora       # 收费版
```

**[Beekeeper](https://www.beekeeperstudio.io/)：数据库管理工具，社区版免费，可以替代Navicat**

```bash
yay -S beekeeper-studio-bin
```

**[Studio 3T](https://studio3t.com/)：MongoDB数据库管理工具，本身免费，收费可以解锁更多功能**

```bash
yay -S studio-3t
```

**[Postman](https://www.postman.com/)：免费的API调试工具，后端开发必备**

```bash
yay -S postman-bin
```

**Go开发环境**

```bash
sudo pacman -S go
```

Go语言的一些常用工具：

```bash
go install github.com/klauspost/asmfmt/cmd/asmfmt@latest
go install github.com/go-delve/delve/cmd/dlv@latest
go install github.com/kisielk/errcheck@latest
go install github.com/davidrjenni/reftools/cmd/fillstruct@latest
go install github.com/rogpeppe/godef@latest
go install golang.org/x/tools/cmd/goimports@latest
go install golang.org/x/lint/golint@latest
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
go install github.com/fatih/gomodifytags@latest
go install golang.org/x/tools/cmd/gorename@latest
go install github.com/jstemmer/gotags@latest
go install golang.org/x/tools/cmd/guru@latest
go install github.com/josharian/impl@latest
go install honnef.co/go/tools/cmd/keyify@latest
go install github.com/fatih/motion@latest
go install github.com/koron/iferr@latest
```

将Go的bin加到$PATH中：

```bash
export PATH=$PATH:$HOME/go/bin
```

**Rust开发环境**

Rust比较特殊，不推荐用`pacman`来安装，建议用`rustup`：

```bash
curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
```

将Rust的bin加到$PATH中：

```bash
export PATH=$PATH:$HOME/.cargo/bin
```

Jetbrains为Rust开发了IDE，[RustRover](https://www.jetbrains.com/rust/)，目前处于功能测试阶段，是免费的，可以抢先体验下。笔者体验下来目前感觉不错：

```bash
yay -S rustrover
```

**Java运行时环境：如果要执行Java程序，可以安装openjdk**

```bash
sudo pacman -S jre-openjdk
sudo pacman -S jdk-openjdk  # 如果需要开发，安装这个，其中包括了jre
```

**[Intellij IDEA](https://www.jetbrains.com/idea/)：Java的王者IDE，社区版是免费的**

```bash
yay -S intellij-idea-community-edition-jre  # 社区免费版
yay -S intellij-idea-ultimate-edition-jre   # 收费版
```

## 显卡驱动

### Intel核显

通过下面命令可以一键安装Intel核显驱动：

```bash
sudo pacman -S mesa lib32-mesa vulkan-intel lib32-vulkan-intel
```

> 注意，只有 Intel HD 4000 及以上的核显才支持 vulkan。

### AMD显卡

> AMD核显和独显方法相同

如果是A卡，需要确认架构是什么才能安装驱动，建议到[List of AMD graphics processing units](https://en.wikipedia.org/wiki/List_of_AMD_graphics_processing_units)进行查询。

下面的表格列出了不同架构的驱动支持：

|   GPU 架构    |    Radeon 显卡    |   开源驱动   | 非开源驱动  |
| :-----------: | :---------------: | :----------: | :---------: |
| GCN 4 及之后  |       多种*       |   AMDGPU*    | AMDGPU PRO* |
|     GCN 3     |       多种        |    AMDGPU    | AMDGPU PRO  |
|     GCN 2     |       多种        | AMDGPU/ ATI* |   不支持    |
|     GCN 1     |       多种        | AMDGPU / ATI |   不支持    |
| TeraScale 2&3 | HD 5000 - HD 6000 |     ATI      |   不支持    |
|  TeraScale 1  | HD 2000 - HD 4000 |     ATI      |   不支持    |
|    旧型号     |   X1000 及之前    |     ATI      |   不支持    |

确认要安装哪个驱动之后，再进行安装，一般来说，A卡建议安装开源驱动：

安装AMDGPU驱动（开源，适用于一些较新的显卡）：

```bash
sudo pacman -S mesa lib32-mesa xf86-video-amdgpu vulkan-radeon lib32-vulkan-radeon
```

安装ATI驱动（开源，适用于比较老的显卡）：

```bash
sudo pacman -S mesa lib32-mesa xf86-video-ati
```

### NVIDIA独显

N卡建议安装闭源驱动，如果你的显卡型号较新，则安装下面的包：

```bash
sudo pacman -S nvidia nvidia-dkms nvidia-settings lib32-nvidia-utils
```

如果是 GeForce 630 以下到 GeForce 400 系列的老卡，安装：

```bash
yay -S nvidia-390xx-dkms nvidia-settings lib32-nvidia-390xx-utils
```

如果是更加老的卡，安装：

```bash
sudo pacman -S mesa lib32-mesa xf86-video-nouveau
```

> ⚠ 警告：特别提示，在ArchLinux玩游戏时N卡体验并不好，建议有游戏需求的还是在Windows下玩游戏。

## 备份和恢复

### Timeshift快照

对于ArchLinux这种滚动升级的操作系统，快照是很重要的，如果滚挂了，可以随时进行回滚操作。timeshift是一个快照管理工具，可以定期对我们的磁盘创建快照。

安装timeshift：

```bash
sudo pacman -S timeshift
```

打开timeshift，第一次启动会自动启动设置向导：

快照类型选择 `BTRFS`：

![timeshift-type](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/timeshift-type.png)

快照位置选择 `BTRFS` 分区：

![timeshift-location](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/timeshift-location.png)

选择快照计划。由于 btrfs 类型快照占用空间相对较小，可以适当提高快照数量。

若希望 `/home` 用户主目录也快照，则勾选在备份中包含 `@home` 子卷：

![timeshift-user](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/timeshift-user.png)

点击 `完成` 结束配置。

然后，就可以非常简单地使用Timeshift图形界面完成快照的创建和回退了：

![timeshift](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/timeshift.png)

### 恢复

ArchLinux激进的滚动更新意味着你的系统随时可能因为滚动更新导致错误。一般的错误可能会导致一些软件无法使用；如果kde出现错误，你可能会无法进入桌面环境；在最严重的情况下，你甚至连系统都进入不了。

首先，如果滚动更新出现错误，应该到[ArchLinux 社区](https://archlinux.org/)中寻求帮助，如果有现成的解决方案，可以直接解决软件问题是最好的。

实在没有办法，或者希望快速恢复时，就需要`Timeshift`上场了。正所谓养兵千日用兵一时，上面我们配置了`Timeshift`快照，就是为了应对这种情况。

如果这时候kde还没有挂掉，我们可以直接进入桌面环境，打开`Timeshift`软件进行恢复。恢复成功之后重启系统即可。

如果连桌面都进入不了，可以通过 `Ctrl` + `Alt` + `F2 ~ F6` 进入 tty 终端。

在终端里面，通过命令进行恢复：

```bash
sudo timeshift --list # 获取快照列表
sudo timeshift --restore --snapshot '20XX-XX-XX_XX-XX-XX' --skip-grub # 选择一个快照进行还原，并跳过 GRUB 安装，一般来说 GRUB 不需要重新安装
```

可能整个系统都挂掉了，无法进入，就是我们俗称的“滚挂了”的情况。

遇到这种情况，先不要慌，把自己事先准备的ArchLinux启动U盘拿出来，重新进入archiso模式。

在这里，通过`arch-chroot`进入坏掉的系统：

```bash
mount -t btrfs -o subvol=/@,compress=zstd /dev/sdxn3 /mnt
mount -t btrfs -o subvol=/@home,compress=zstd /dev/sdxn3 /mnt/home
mount /dev/sdxn1 /mnt/boot/efi
swapon /dev/sdxn2
arch-chroot /mnt
```

然后进行恢复：

```bash
sudo timeshift --restore --snapshot-device /dev/sdx
sudo timeshift --list # 获取快照列表
sudo timeshift --restore --snapshot '20XX-XX-XX_XX-XX-XX' --skip-grub # 恢复
```

无法进入系统很有可能是grub坏了，如果你觉得你的grub有问题，可以在恢复的时候去掉`--skip-grub`选项，以重新安装grub。

另外，也可以通过下面的方式手动重新安装grub：

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ARCH
grub-mkconfig -o /boot/grub/grub.cfg  # 重新生成配置文件
```

总之，身边一定要准备一个archiso，或是其他形式的Live CD，能够在系统本身无法进入时有手段进行抢修。

**不建议动不动就重装，archiso重装grub可以解决大部分系统无法进入的问题。**

## 其他软件

这里推荐一些其它日常中可能会用到的软件，仅供参考。

[Flameshot](https://github.com/flameshot-org/flameshot) - 强大但简单易用的屏幕截图软件。截图后可以进行快捷的编辑。

```bash
sudo pacman -S flameshot
```

[WPS](https://www.wps.cn/) - 国产软件里面少有的原生支持Linux的，如果你有在Linux下编辑Office的需求，可以安装：

```bash
yay -S wps-office
```

[LibreOffice](https://zh-cn.libreoffice.org/) - 如果不想用WPS专有软件，自由开源的LibreOffice很适合你：

```bash
sudo pacman -S libreoffice-still libreoffice-still-zh-cn  # 正式版
sudo pacman -S libreoffice-fresh libreoffice-fresh-zh-cn  # 尝鲜版
```

[Telegram](https://aur.archlinux.org/packages/telegram-desktop-bin) - Telegram（电报）是跨平台的即时通信软件。其客户端是自由软件（桌面端在 [GPLv3open](https://github.com/telegramdesktop/tdesktop/blob/dev/LICENSE) 协议下发布），但服务器是专有软件。

```bash
sudo pacman -S telegram-desktop
```

[Linux QQ](https://im.qq.com/linuxqq/index.shtml) - 腾讯基于electron做的跨平台版本QQ

```bash
yay -S linuxqq
```

> ArchLinux下的微信还没有electron版，只有基于wine的版本（有一个Deepin主导的星火商店版本`com.qq.weixin.spark`）。个人认为体验不好（各种字体、缩放问题），因此这里不做推荐。

[KMail](https://archlinux.org/packages/extra/x86_64/kmail/) - 先进的电子邮件客户端，能与 GMail 等常用电子邮件服务提供商进行整合。KMail 支持各种电子邮件协议，包括 POP3、IMAP、Microsoft Exchange（EWS）等。

```bash
sudo pacman -S kmail
```

[kcalc](https://archlinux.org/packages/extra/x86_64/kcalc/) - 科学计算器

```bash
sudo pacman -S kcalc
```

[Okular](https://archlinux.org/packages/extra/x86_64/okular/) - KDE 开发的一款功能丰富、轻巧快速的跨平台文档阅读器。可以使用它来阅读 PDF 文档、漫画电子书、Epub 电子书，浏览图像，显示 Markdown 文档等。

```bash
sudo pacman -S okular
```

[YesPlayMusic](https://github.com/qier222/YesPlayMusic) - 高颜值的第三方网易云播放器。

```bash
yay -S yesplaymusic
```

[Listen 1](https://github.com/listen1/listen1_desktop) - Listen 1 作为“老牌”的听歌软件可以搜索和播放来自网易云音乐、虾米、QQ 音乐、酷狗音乐、酷我音乐、Bilibili、咪咕音乐网站的歌曲，让你的曲库更全面。

```bash
yay -S listen1-desktop-appimage
```

## 系统美化

系统美化永远不是一件重要的事情。一个Linux系统应该优先是好用的，稳定的，其次才是美化。因此我将美化相关内容放在最后一节进行。

> ⚠️警告：KDE过度美化可能会造成系统不稳定或出现奇怪的错误，在美化前强烈建议使用[Timeshift创建备份](#备份和恢复)，以在出现问题之后能够快速回滚到之前的状态。

每个人的审美不同，对于美化的界定也不同，这里只给出基于我自己审美的美化步骤作为参考，每个人的ArchLinux都应该是不一样的！

### 系统图标

我们可以更改默认的系统图标，推荐[Tela icon theme](https://github.com/vinceliuice/Tela-icon-theme)，安装：

```bash
sudo pacman -S tela-icon-theme-git
```

在设置页面直接更改即可：

![icon](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/icon.png)

### 全局主题

推荐[Layan](https://github.com/vinceliuice/Layan-gtk-theme)主题，这是一个来自中国的设计师设计的主题，还是比较好看的。

主题配合Kvantum Manager可以达到更好的效果：

```bash
sudo pacman -S kvantum
```

在[这里](https://www.pling.com/p/1325246/)下载Layan kvantum主题，并解压。

打开Kvantum Manager，选择主题并安装，接下来在`Change/Delete Theme`中选择Layan，`Use this theme`：

![Kvantum](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/layan_kvantum.png)

然后在KDE系统设置中的`Application Style`中选择`Kvantum`：

![kde-app-style](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/layan_app_style.png)

安装Layan全局主题：

```bash
yay -S plasma5-themes-layan-git
```

设置全局主题为Layan：

![layan-theme](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/layan_theme.png)

重启电脑，让主题生效。

### 引导页面

开机时有个漂亮的GRUB也是很舒服的。

在[这里](https://www.pling.com/p/1482847/)下载Distro的GRUB主题并解压。接下来cd进解压出来的文件夹，输入命令：

```bash
sudo cp . /usr/share/grub/themes/Distro -rf
```

以将主题放置在系统的GRUB默认文件夹内。

接着编辑`/etc/default/grub`文件，找到`#GRUB_THEME=`一行，将前面的注释去掉，并指向主题的`theme.txt`文件。即

```bash
#GRUB_THEME=
GRUB_THEME="/usr/share/grub/themes/Distro/theme.txt" #修改后
```

然后再在终端输入：

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

这样，引导页面美化就完成了。

### SDDM主题

安装[McSur](https://github.com/yeyushengfan258/McSur-kde)主题：

```bash
yay -S plasma5-theme-mcsur-git
```

设置SDDM主题：

![sddm](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/sddm.png)

### Dock布局

我们将通过KDE自带的编辑功能实现类似MacOS的Dock布局，注意我没有使用[latte-dock](https://github.com/KDE/latte-dock)来实现，因为笔者实际用下来发现latte有太多难以启齿的Bug，并且需要吃掉一些性能。如果想使用Latte请自行参考文档。

右键菜单栏，进入编辑模式，将其移动到顶部，并略微修改布局：

![menu](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/menu.png)

在桌面，编辑组件，选择`Add Panel`，类型选择`Default Panel`。

移除除了`Task Manager`以外的小组件，并通过增加`Panel Spacer`让任务管理器居中，效果类似于：

![edit-dock](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/dock.png)

点击`More Options`，在`Visbility`选择`Windows Can Cover`，以让Dock栏实现自动隐藏。

最终效果：

![desktop](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2024-01-04-archlinux-install/desktop.png)

恭喜，到这里，就完成了整个ArchLinux的安装和美化工作。