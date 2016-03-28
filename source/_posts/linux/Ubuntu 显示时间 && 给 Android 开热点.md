title: "Ubuntu 显示时间 && 给 Android 开热点"
date: 2015-04-02 16:50:00
categories: Linux
---

转到 Linux 下之后最不习惯的就是没热点用，还有时不时抽风的时间显示。
所以这两天特地搜了一下解决方案搞定了它。
<!--more-->

### 菜单栏不显示时间解决方法

首先我们用下面的命令来确认一下是否安装了日期时间指示器：

    sudo apt-get install indicator-datetime

如果确认已经安装，那么我们需要重新配置它：

    sudo dpkg-reconfigure --frontend noninteractive tzdata

然后重启Unity：

    sudo killall unity-panel-service

经过以上步骤之后，时间就会显示在状态栏了。

---

### 给 Android 手机开热点

昨天用系统自带的 Ad-hoc 尝试开热点，但是 Android 系统不支持 Ad-hoc 。
今天发现 Android 手机支持 Ap-hotspot ，就又尝试了一番。

命令如下：

    //添加含有ap-hotspot的资源
    sudo add-apt-repository ppa:nilarimogard/webupd8
    //更新资源
    sudo apt-get update
    //如果之前安装了 ap-hotspot 或者 hostapd 的需要卸载
    sudo apt-get remove hostapd
    //在 32 位系统中安装没有 bug 的 hostapd 版本（64 位： i386 → amd64）
    cd /tmp
    wget http://old-releases.ubuntu.com/ubuntu/pool/universe/w/wpa/hostapd_1.0-3ubuntu2.1_i386.deb
    sudo dpkg -i hostapd*.deb
    //防止 hostapd 升级到有问题的版本
    sudo apt-mark hold hostapd
    //安装ap-hotspot
    sudo apt-get install ap-hotspot
    //配置
    sudo ap-hotspot configure
    //关闭防火墙
    sudo ufw disable
    //启动服务，关闭对应 stop 
    sudo ap-hotspot start

在不用无线网卡时开个热点爽爽的，写成超级捞的脚本（这也能叫脚本...）：

    #!\bin\bash
    sudo ap-hotspot start