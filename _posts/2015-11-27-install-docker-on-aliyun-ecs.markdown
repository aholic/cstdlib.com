---
layout: post
title:  "在阿里云上装docker"
date:   2015-11-27 08:42:35
categories: tech
keywords: "阿里云 ECS docker 安装"
description: "记录在阿里云ECS上安装docker的过程"
---

准备在阿里云的ECS上装的docker，我的主机是64位的Ubuntu14.04，内核版本是3.13，如下所示。
    
    $uname -a
    Linux iZ258qfn3t4Z 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
    
安装过程基本是按照这个[步骤](http://www.docker.org.cn/book/install/26_install-docker-trusty-14.04.html)来的。
它所述的安装过程其实很简单，依次这么几步骤：

1. 下载docker安装脚本并执行（假设你有curl或者wget）

        $curl -sSL https://get.docker.com/ | sh

2. 启动docker

        $sudo docker daemon

    或者

        $sudo service docker start

3. 验证安装

        $sudo docker run hello-world

但是安装的过程中遇到这么几个问题，在这里记录一下。

1. 找不到可用的内网IP
    
    启动docker daemon的过程中出现了这么一个问题：

        FATA[0000] Error starting daemon: Error initializing network controller: Error creating default "bridge" network: failed to parse pool request for address space "LocalDefault" pool "" subpool "": could not find an available predefined network

    大致意思就是找不到一段可用的IP吧，后来查到的问题是因为阿里云上默认把几个内网的网段都路由了。
    在`/etc/network/interfaces`里面是这么写的。
    
        auto lo
        iface lo inet loopback
        auto eth1
        iface eth1 inet static
        address xxx.xxx.xxx.xxx
        netmask xxx.xxx.xxx.xxx
        up route add -net 0.0.0.0 netmask 0.0.0.0 gw xxx.xxx.xxx.xxx dev eth1
        auto eth0
        iface eth0 inet static
        address xxx.xxx.xxx.xxx
        netmask xxx.xxx.xxx.xxx
        up route add -net 192.168.0.0 netmask 255.255.0.0 gw xxx.xxx.xxx.xxx dev eth0
        up route add -net 172.16.0.0 netmask 255.240.0.0 gw xxx.xxx.xxx.xxx dev eth0
        up route add -net 100.64.0.0 netmask 255.192.0.0 gw xxx.xxx.xxx.xxx dev eth0
        up route add -net 10.0.0.0 netmask 255.0.0.0 gw xxx.xxx.xxx.xxx eth0
        
    里面的xxx.xxx.xxx.xxx已打码，并不影响你继续看本页面。最后几行的`up route add`开头的配置是说在接口启用的时候添加一条路由规则，
    比如`up route add -net 172.16.0.0 netmask 255.240.0.0 gw xxx.xxx.xxx.xxx dev eth0`就是把`172.16.0.0/20`这个网段都路由到网关xxx.xxx.xxx.xxx去。
    所以我们要把它解放出来，让docker用。
    
    编辑`/etc/network/interfaces`文件，用#注释掉`up route add -net 172.16.0.0 netmask 255.240.0.0 gw xxx.xxx.xxx.xxx dev eth0`，
    然后执行`route del -net 172.16.0.0 netmask 255.240.0.0`把现在启用的路由也删掉。可以用`route -n`看一下现在的路由表。
    
2. 内核不支持swap内存限制

    解决了上一个问题继续启动docker daemon又会遇到这样一个问题：

        WARN[0000] Your kernel does not support swap memory limit.

    这也写了，是个warning，所以我估计不解决docker daemon也能跑起来，但是warning就是让人不舒服。这日志也说得很明白了，不支持swap内存限制。
    改一下内核参数，Ubuntu默认使用grub，在文件`/etc/default/grub`里面，把`GRUB_CMDLINE_LINUX=""`改成`GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"`，
    然后执行`sudo update-grub`更新一下，然后重启机器。
    
    重启后可以用`cat /proc/cmdline`查看刚才的选项是不是启用了
    
    
