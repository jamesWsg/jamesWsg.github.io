---
layout: post
title: windows远程桌面连接ubuntu desktop
category: 技术
---




# 背景

git仓库放在远端服务器，希望在远端的ubuntu服务器上调试代码，本地是window环境。

# server 端



## ubuntu 18.04 配置

需要安装 

```
sudo apt install xrdp xorgxrdp






############## 备注
我的环境 安装时 选择了图形环境，所以 下面的图形包 已经包括了
sudo apt install xserver-xorg-core

注意，ubuntu 发行版自带的的 xserver 是如下名字
wsg@wsgCode ~ $ dpkg -l |grep -Ei xserver
ii  xserver-xorg-core-hwe-18.04                2:1.20.8-2ubuntu2.2~18.04.3                      amd64        Xorg X server - core server


```





```
wsg@wsgCode ~ $ sudo apt-get install xorgxrdp-hwe-18.04
[sudo] password for wsg: 
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following packages were automatically installed and are no longer required:
  conntrack cri-tools ebtables ethtool kubectl libllvm7 libllvm9 linux-hwe-5.4-headers-5.4.0-42 openjdk-11-jdk-headless socat
Use 'sudo apt autoremove' to remove them.
Recommended packages:
  xorg-hwe-18.04
The following NEW packages will be installed:
  xorgxrdp-hwe-18.04
0 upgraded, 1 newly installed, 0 to remove and 339 not upgraded.
Need to get 79.4 kB of archives.
After this operation, 455 kB of additional disk space will be used.
Get:1 http://mirrors.aliyun.com/ubuntu bionic-updates/universe amd64 xorgxrdp-hwe-18.04 amd64 0.9.5-2~18.04.1 [79.4 kB]
Fetched 79.4 kB in 0s (306 kB/s)              
Selecting previously unselected package xorgxrdp-hwe-18.04.
(Reading database ... 250642 files and directories currently installed.)
Preparing to unpack .../xorgxrdp-hwe-18.04_0.9.5-2~18.04.1_amd64.deb ...
Unpacking xorgxrdp-hwe-18.04 (0.9.5-2~18.04.1) ...
Setting up xorgxrdp-hwe-18.04 (0.9.5-2~18.04.1) ...

```



> 注意，必须安装 xorgxrdp,一开始没有 安装 该包，导致 连接一直





## xrdp 配置



1, 修改 xrdp的 startwm.sh

```
wsg@wsgCode ~ $ vim /etc/xrdp/startwm.sh 

#test -x /etc/X11/Xsession && exec /etc/X11/Xsession
#exec /bin/sh /etc/X11/Xsession
gnome-session



```



2, 增加 配置(否则 mstsc 后，会提示 "Authentication is Required to create a color managed device")

````
wsg@wsgCode ~ $ cat /etc/polkit-1/localauthority.conf.d/02-allow-colord.conf 
polkit.addRule(function(action, subject) {
 if ((action.id == "org.freedesktop.color-manager.create-device" ||
 action.id == "org.freedesktop.color-manager.create-profile" ||
 action.id == "org.freedesktop.color-manager.delete-device" ||
 action.id == "org.freedesktop.color-manager.delete-profile" ||
 action.id == "org.freedesktop.color-manager.modify-device" ||
 action.id == "org.freedesktop.color-manager.modify-profile") &&
 subject.isInGroup("{users}")) {
 return polkit.Result.YES;
 }
 });
wsg@wsgCode ~ $ 

````



# windows client端

无需特殊配置，直接通过 mstsc 用 ubuntu的ssh 账号登陆即可

# 故障情况



## 连接后蓝屏报错

![image-20211114165048758](../img/windows-mstsc-xrdp/err1.png)



```
wsg@wsgCode ~ $ vim .xorgxrdp.10.log

Fatal server error:
[   480.939] (EE) parse_vt_settings: Cannot open /dev/tty0 (Permission denied)
[   480.939] (EE) 
[   480.939] (EE) 
Please consult the The X.Org Foundation support 
         at http://wiki.x.org
 for help. 
[   480.939] (EE) Please also check the log file at ".xorgxrdp.10.log" for additional information.
[   480.939] (EE) 

```



### xrdp 进程

```
wsg@wsgCode /var/run/xrdp $ ll
total 8
drwxr-xr-x  3 xrdp xrdp  100 11月 14 16:33 ./
drwxr-xr-x 39 root root 1160 11月 14 17:44 ../
drwxrwsrwt  2 xrdp xrdp   40 11月 14 16:41 sockdir/
-rw-------  1 xrdp xrdp    4 11月 14 16:33 xrdp.pid
-rw-------  1 xrdp xrdp    4 11月 14 16:33 xrdp-sesman.pid


```



### 相关log 报错

```
wsg@k8snode1v18:~$ grep -Ei err /var/log/xrdp.log 
[20211114-18:04:08] [ERROR] Cannot read private key file /etc/xrdp/key.pem: Permission denied
[20211114-18:04:11] [ERROR] Cannot read private key file /etc/xrdp/key.pem: Permission denied

wsg@k8snode1v18:~$ ll  /etc/xrdp/key.pem
lrwxrwxrwx 1 root root 38 11月 14 18:01 /etc/xrdp/key.pem -> /etc/ssl/private/ssl-cert-snakeoil.key


wsg@k8snode1v18:~$ sudo ls -l /etc/ssl/private/ssl-cert-snakeoil.key
-rw-r----- 1 root ssl-cert 1704 8月   3  2020 /etc/ssl/private/ssl-cert-snakeoil.key


```

> 上面这个 报错，没有关系， 不用处理。 我在 73.53 上 有处理过。

### xrdp 的配置

```
wsg@k8snode1v18:/etc/xrdp$ ll
total 304
drwxr-xr-x   3 root root  4096 11月 14 18:01 ./
drwxr-xr-x 133 root root 12288 11月 14 18:01 ../
lrwxrwxrwx   1 root root    36 11月 14 18:01 cert.pem -> /etc/ssl/certs/ssl-cert-snakeoil.pem
lrwxrwxrwx   1 root root    38 11月 14 18:01 key.pem -> /etc/ssl/private/ssl-cert-snakeoil.key




```

