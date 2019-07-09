---
layout: post
title: linux UEFI problem handle
category: 技术
---



# 背景

延续前面的blog（linux installer kernel patch），在解决了Gigabyte设备无法安装OS的问题后，发现有些情况下安装完成后，却无法正常启动（准确说无法找到引导设备，无法看到grub引导界面）。正好也印证了一句老话，好事多磨，要想顺利做完一件事情，要么会出奇的顺利，要么会一波三折。所以，当你顺利的完成事情时，保持感恩的心态；当你遇到问题时，也请做好准备，迎接下一波问题的到来。

牢骚就这么多，处理过程整理如下：



# 处理过程

安装完成后，在发现无法正常进入grub引导界面，基本定位问题在主板的firmware和引导分区的配合上，主板firmware和引导相关的配置主要是 CSM模式（legacy，UEFI，UEFI和legacy）和 boot order。验证过boot order的设置没有问题（确实将安装系统的盘 指定为 第一个 引导设备）。

然后将CSM模式设置为legacy，安装OS 后，发现可以正常引导。但是设置成UEFI后，安装OS后，就是无法进入grub引导界面。所以基本将问题确定为 我们的系统和gigabyte的主板在 UEFI模式下，匹配有问题。（我们的系统在和超微的主板做同样的测试，发现没有问题，怀疑不同的主板厂商，在落地UEFI规范时，可能会有差别）

确定好问题后，我们尝试将ubuntu18.04安装在gigabye设备上，奇迹的发现竟然可以正常进入grub界面，因为我们的系统是基于ubuntu14，看起来高版本做了相应的优化。

安装好的ubuntu18.04的boot分区 目录结构如下：

```
/EFI
--/BOOT
  --/BOOTx64.efi
--/fedora
  --/xxx.efi
  --/xxx.efi
  -- ...
```



通过ISO 的rescue bromken system选项，可以手动去挂载我们系统所在盘的第一个分区，去对比和ubuntu18.04的异同情况，发现我们系统的boot分区的目录结构如下：

```
/EFI
--/ubuntu
  --/xxx.efi
  --/xxx.efi
  -- ...
```

对比发现，我们安装好的系统少了一个 BOOT的目录，网上有资料说明BOOT目录在一些特殊的设备上会有帮助

```
debian帮助文档中提到，一些机器的EFI可能存在bug：
Weak EFI implementation only recognizes the fallback bootloader
某些比较脆弱的EFI实现，并不能认识我们的bootloader，导致启动的时候，他会查找默认的回退路径。
UEFI 规范定义了一种“回退”路径 (Fallback path)，用于启动此类启动管理器项，其工作原理类似于 BIOS 驱动器启动：它会在标准位置查找某些启动装载程序代码。但是其中的细节和 BIOS 不同。当尝试以这种方式启动时，固件真正执行的操作相当简单。固件会遍历磁盘上的每个 EFI 系统分区（按照磁盘上的分区顺序）。在 ESP 内，固件将查找位于特定位置的具有特定名称的文件。在 x86-64 PC 上，固件会查找文件\EFI\BOOT\BOOTx64.EFI。固件实际查找的是 \EFI\BOOT\BOOT{计算机类型简称}.EFI，其中，“x64”是 x86-64 PC的“计算机类型简称”。文件名还有可能是 BOOTIA32.EFI (x86-32)、BOOTIA64.EFI (Itanium)、BOOTARM.EFI（AArch32，即32位ARM）BOOTAA64.EFI（AArch64，即64位ARM）。然后，固件将执行找到的第一个有效文件（当然，文件需要符合UEFI规范中定义的可执行格式）。
```

回到我们的情况，gigabyte的机器可能只认 /boot/efi/EFI/BOOT/BOOTx64.EFI 这个回退路径，因此我们尝试 自己的文件 /boot/efi/EFI/grub/grubx64.efi 拷贝到了 回退路径，并且重命名为BOOTx64.EFI， 这样，之前不能启动的路径就可以自如的启动了。



# 总结

新的服务器都开始采用UEFI的引导方式，取代旧有的BIOS（MBR）的方式，一方面UEFI提供了更丰富的引导特性，另一方面突破了传统BIOS方式的2TB磁盘空间的限制。







#参考文档

> <http://jcf94.com/2017/08/06/2017-08-06-efi/>
>
> <https://www.happyassassin.net/2014/01/25/uefi-boot-how-does-that-actually-work-then/>
>
> 