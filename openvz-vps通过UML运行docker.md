# OpenVZ VPS 通过 UML 运行 Docker

**系统环境**

openvz vps: 内核 linux 2.6.32-042stab112.15，可用硬盘空间 15G, 内存 768M, 操作系统 Ubuntu 14.04.4 LTS

因为想利用闲置的 openvz vps 运行 gitlab runner 来执行 ci 测试，而 openvz 的内核限制又很多比如运行 docker 需要 linux kerel 3.10以上，而 openvz 又不支持修改内核，所以考虑使用 UML (User Mode Linux) 在 openvz vps 上安装一个 archlinux 系统，再在 archlinux 系统里跑 docker。


## 准备工作


**编译UML内核文件vmlinux**


在 [www.kernel.org](https://www.kernel.org) 选择当前最新的稳定版内核下载并编译 [linux-4.8.15.tar.xz](https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.8.15.tar.xz)


openvz 机器配额比较低，所以我选择在另一台配置较高的机器编译内核。

```shell
wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.8.15.tar.xz
tar xf linux-4.8.15.tar.xz
cd linux-4.8.15
```

下载我的内核配置文件

```
wget https://raw.githubusercontent.com/xdtianyu/Docs/master/kernel-4.8.15-uml.config -O .config
```

运行 `menuconfig`，按你的需要修改，按两次 `ESC` 退出

```
make ARCH=um menuconfig
```

编译内核文件 `vmlinux`

```
make ARCH=um vmlinux -j4
```

编译内核模块文件

```
make ARCH=um modules -j4
```

将生成的模块和内核文件安装/拷贝出来，如复制到上一层目录 `../uml` 文件夹内并打包

```
make ARCH=um modules_install INSTALL_MOD_PATH=../uml
cp vmlinux ../uml

cd ..
tar czf uml.tar.gz uml
```

之后将 `uml.tar.gz` 文件上传到目标 `openvz vps` 即可， 通过执行 `vmlinux` 内核文件，我们就可以在 `openvz` 机器上安装/运行一个完整的 `archlinux` 系统了。

更多关于编译内核的内容请参考 [https://www.kernel.org/doc/makehelp.txt](https://www.kernel.org/doc/makehelp.txt)


## 安装系统

首先在 `openvz` 机器上解压上传的 `uml.tar.gz` 文件，将 `vmlinux` 移动到 `/usr/bin` 目录下

```
tar xf uml.tar.gz
cp uml/vmlinux /usr/bin
```

建议找一个可用空间够大的目录存放要安装的系统硬盘镜像， 如 `/home/uml` 目录

```
mkdir /home/uml
mv uml.tar.gz /home/uml
mkdir arch
cd arch
```

**创建 archlinux rootfs环境**

下边列出安装系统的必要步骤

```shell
wget http://mirror.rackspace.com/archlinux/iso/latest/archlinux-bootstrap-2016.12.01-x86_64.tar.gz
tar xzf archlinux-bootstrap-2016.12.01-x86_64.tar.gz
mv root.x86_64 root

vi root/etc/resolv.conf
# 加上一行 nameserver 8.8.8.8，保存

mount --rbind /proc root/proc
mount --rbind /sys root/sys
mount --rbind /dev root/dev
mount -t tmpfs tmpfs root/tmp
mount --rbind /root root/root

vi root/etc/pacman.d/mirrorlist
# 搜索离你最近的国家镜像，去掉对应的注释

root/bin/arch-chroot root /bin/bash

# 此时已经进入 chroot 环境
pacman-key --init
pacman-key --populate archlinux

# 安装基本系统
pacman -Sy base

# 安装一些常用工具，可以跳过这一步
pacman -Sy openssh vim net-tools dnsutils inetutils iproute2 sudo wget git python-pip dmidecode dstat

# 改控制台，也可以不改，通过 screen 连接 pts
systemctl enable getty@tty0
systemctl disable getty@tty1

# 退出 chroot
exit

# 解除 rbind
umount root/{dev,proc,sys,tmp} -l
```

注意如果出现 `unshare: unrecognized option '--fork'` 错误，请更新 `util-linux`, 参考 [http://askubuntu.com/a/586164/477390](http://askubuntu.com/a/586164/477390)

此时在 `root` 文件夹下就有了完整的 `rootfs`，可以启动 `UML` 了。


**设置网络**


首先在主机商的面板（SolusVM）打开 TUN/TAP 功能。一般能做 VPN（PPTP、L2TP等） 的 VPS 都有这个选项。之后配置 TAP 设备：

```
ip tuntap add tap0 mode tap
ip addr add 10.0.0.1/24 dev tap0
ip route add default via 10.0.0.1 dev venet0:0
ip link set tap0 up
iptables -P FORWARD ACCEPT
iptables -t nat -A POSTROUTING -o venet0 -j MASQUERADE
iptables -t nat -A POSTROUTING -o venet0:0 -j MASQUERADE
iptables -t nat -A POSTROUTING -o tap0 -j MASQUERADE
```

**安装 archlinux 到镜像文件**


安装 `e2fsprogs`， 创建一个 10G 镜像文件并格式化为 `ext4` 文件系统，也可以格式化为其他 linux 文件系统。

```
fallocate -l 10G arch.img
mkfs.ext4 arch.img
```

从 `rootfs` 启动 `UML`

```
vmlinux root=/dev/root rootfstype=hostfs hostfs=./root ubd0=arch.img eth0=tuntap,tap0 mem=256m 
```

`mem` 是内存大小，此处取 256M（可以根据你的内存情况修改）。输入用户名 `root`，密码为空，接下来安装系统

```
# 配置网络
ip link set eth0 up  
ip addr add 10.0.0.2/24 dev eth0  
ip route add default via 10.0.0.1 dev eth0

# 安装基础系统文件
mount /dev/ubda /mnt
mkdir -p /mnt/var/lib/pacman
pacman -Sy base -r /mnt
pacman -Sy haveged -r /mnt # entropy 生成器

# 安装一些常用工具，可以跳过这一步
pacman -Sy openssh vim net-tools dnsutils inetutils iproute2 sudo wget git python-pip dmidecode dstat -r /mnt

mount --rbind /proc /mnt/proc
mount --rbind /sys /mnt/sys
mount --rbind /dev /mnt/dev
mount -t tmpfs tmpfs /mnt/tmp
mount --rbind /root /mnt/root

chroot /mnt /bin/bash
nano /etc/pacman.d/mirrorlist # mirrorlist
```

编辑网络配置

```
nano /etc/systemd/network/50-static.network 
```

添加

```
[Match]
Name=eth0

[Network]
Address=10.0.0.2/24  
Gateway=10.0.0.1 
```

启用各项服务，配置 时区、locale 等

```
nano /etc/resolv.conf
# 加上一行 nameserver 8.8.8.8，保存

systemctl enable systemd-networkd
systemctl enable getty@tty0
systemctl disable getty@tty1

ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
sed -i 's/#en_US.UTF/en_US.UTF/' /etc/locale.gen
locale-gen
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
echo 'um-arch' > /etc/hostname # 配置主机名
nano /etc/hosts # 加入 127.0.1.1 um-arch.localdomain um-arch

exit

genfstab -U /mnt >> /mnt/etc/fstab

halt # 关闭 UML
```

## 启动 archlinux

```
vmlinux root=/dev/ubda ubd0=/home/uml/arch/arch.img eth0=tuntap,tap0 mem=256m
```

登录用户名 `root`， 密码为空。

```
systemctl start haveged
pacman-key --init
pacman-key --populate archlinux

pacman -Syu
```

添加一些 alias

```
vi /etc/bash.bashrc
```

最后增加

```
alias ls='ls --color=auto'
alias vi='vim'
alias crontab='fcrontab'
export EDITOR='vim'
```

```
source /etc/bash.bashrc
```

创建新用户及设置密码，启动 ssh 服务

```
passwd root

useradd YOUR_USERNAME -m
passwd YOUR_USERNAME

systemctl enable sshd
systemctl start sshd
```

之后可以在另一个终端中 ssh 登录

```
ssh YOUR_USERNAME@10.0.0.2
```

关闭 uml 命令 `halt`即关机，要保持后台运行可以将 `uml` 在 `screen` 中开机，之后 `CTRL-A-D` 退出到后台。

至此 `uml` 就安装完成了，可以在 `uml` 系统中安装 `docker`，`gitlab-runner` 等来将 `openvz vps` 利用起来。


## 拷贝内核模块

将 `uml` 启动到后台后，通过 `scp` 命令拷贝内核模块文件到 `uml` 系统并安装在 `/lib/modules` 目录。

```
scp /home/uml/uml.tar.gz YOUR_USERNAME@10.0.0.2:/home/YOUR_USERNAME
ssh YOUR_USERNAME@10.0.0.2

su
tar xf uml.tar.gz

mv uml/lib/modules/4.8.15/ /lib/modules
```

## 添加 swapfile 文件

```
cd /
dd if=/dev/zero of=swapfile bs=1M count=1024
chown 0600 swapfile
mkswap swapfile
```

修改 `/etc/fstab`， 添加 `swap` 挂载，最后一行添加

```
/swapfile none swap sw 0 0
```

执行 `swapon /swapfile` 启用 `swap` 或者重启 `uml` 系统来启用。


## 安装 docker

```
pacman -S docker
systemctl enable docker
systemctl start docker
```

## 安装 gitlab-runner

```
wget -O /usr/local/bin/gitlab-ci-multi-runner https://gitlab-ci-multi-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-ci-multi-runner-linux-amd64
chmod +x /usr/local/bin/gitlab-ci-multi-runner
```

创建 `gitlab-runner` 用户

```
useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
```

注册到私有 `gitlab` 服务

```
gitlab-ci-multi-runner register

gitlab-ci-multi-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
gitlab-ci-multi-runner start
```

注册成功后即可在 `gitlab` 后台查看并运行 `gitlab-runner` 服务。经过测试，内存较低时使用 `openvz uml gitlab docker runner` 效率非常低，请慎重使用。

`uml` 提供了一种完整的 `linux` 系统支持，可以运行一些有趣的服务，之后再进行端口转发，可以充分利用 `openvz` 廉价的特点，并规避其不宜扩展的缺点，实现高可定制性的需求。


## 参考链接

[使用UML合租VPS](https://typeblog.net/how-did-i-share-aliyun/)

[OpenVZ VPS 安装 User-mode Linux 以实现 BBR 拥塞控制](https://blog.amayume.net/openvz-vps-an-zhuang-user-mode-linux-yi-shi-xian-bbr-yong-sai-kong-zhi/)
