# kvm 虚拟机安装 mac

参考 [https://github.com/kholia/OSX-KVM](https://github.com/kholia/OSX-KVM)

宿主系统环境 ArchLinux, libvirtd 3.0.0, QEMU emulator version 2.8.0

## 安装过程

**1\.下载安装镜像文件**

下载地址 [http://bit.do/bootable](http://bit.do/bootable)，来源 [issuecomment-252537393](https://github.com/kholia/OSX-KVM/issues/21#issuecomment-252537393)

建议在服务器上使用 [megatools](https://github.com/megous/megatools) 下载 `Install_OS_X_10.11.6_El_Capitan.iso` 文件保存到 `/home/libvirt/boot/` 目录。

**2\.创建硬盘镜像文件**

```shell
qemu-img create -f qcow2 /home/libvirt/images/mac2.img 64G
```

**3\.下载 mac 引导文件，导入 mac.xml qemu 定义文件**

```shell
wget https://raw.githubusercontent.com/kholia/OSX-KVM/master/enoch_rev2839_boot -O /home/libvirt/boot/mac_enoch_rev2839_boot
wget https://gist.githubusercontent.com/xdtianyu/b871b8dde51522caeda001c484f1e48e/raw -O mac2.xml
```

```shell
virsh define mac2.xml
```

**4\.启动 mac2 虚拟机**

取消 `table` 鼠标类型的支持

```shell
virsh edit mac2
```

移除下面内容，注意安装完成后再安装鼠标驱动，会再添加下面的内容。

```xml
    <input type='tablet' bus='usb'>
      <address type='usb' bus='0' port='1'/>
    </input>
```
修改 `<input type='mouse' bus='ps2'/>` 为 `<input type='mouse' bus='usb'/>`

启动虚拟机

```shell
virsh start mac2
```

使用 webvirtmgr 的 novnc 安装系统，或者使用 `netstat -natp|grep 590` 命令查看 mac2 vnc 监听的端口，使用 vnc 远程工具安装系统。

![mac1](https://github.com/xdtianyu/Docs/raw/master/art/mac1.png)

敲击回车，等待引导加载完成后进入安装界面。

![mac2](https://github.com/xdtianyu/Docs/raw/master/art/mac2.png)

这时候如果虚拟机内的鼠标和外部的不对应有延时，可以使用快速移动鼠标，利用虚拟机内的鼠标惯性移动到目标位置。安装完成后会安装鼠标驱动修复这个问题。

按TAB移动选择项，按空格选择项目。

![mac5](https://github.com/xdtianyu/Docs/raw/master/art/mac5.png)

使用硬盘工具对虚拟机硬盘进行分区

![mac7](https://github.com/xdtianyu/Docs/raw/master/art/mac7.png)

分区完成后打开终端，复制文件 

```shell
cp -av /Extra /Volumes/KVMDisk
```
![mac8](https://github.com/xdtianyu/Docs/blob/master/art/mac8.png)

![mac9](https://github.com/xdtianyu/Docs/blob/master/art/mac9.png)

在菜单栏中退出终端后会弹出之前的安装界面，选择安装到 `KVMDisk`，等待安装完成。

![mac11](https://github.com/xdtianyu/Docs/raw/master/art/mac11.png)

![mac12](https://github.com/xdtianyu/Docs/raw/master/art/mac12.png)

![mac13](https://github.com/xdtianyu/Docs/raw/master/art/mac13.png)

![mac14](https://github.com/xdtianyu/Docs/raw/master/art/mac14.png)

![mac15](https://github.com/xdtianyu/Docs/raw/master/art/mac15.png)

![mac16](https://github.com/xdtianyu/Docs/raw/master/art/mac16.png)

**5\.安装 tablet-usb 驱动**

参考 [http://philjordan.eu/osx-virt/](http://philjordan.eu/osx-virt/)

在 mac 虚拟机里打开 `http://philjordan.eu/osx-virt/` 并下载 `http://philjordan.eu/osx-virt/binaries/QemuUSBTablet-1.2.pkg` 文件安装。

![mac18](https://github.com/xdtianyu/Docs/raw/master/art/mac18.png)

![mac19](https://github.com/xdtianyu/Docs/raw/master/art/mac19.png)

关闭虚拟机，修改 xml 文件

```shell
virsh destroy mac2
virsh edit mac2
```

修改 `<input type='mouse' bus='usb'>` 为 ` <input type='tablet' bus='usb'>`

启动虚拟机

```
virsh start mac2
```

**6\.修改分辨率**

修改 `/Extra/org.chameleon.boot.plist` 文件，`</dict>`行上添加如下内容，重启虚拟机系统。

```
<key>Graphics Mode</key>
<string>1440x900x32</string>
```
