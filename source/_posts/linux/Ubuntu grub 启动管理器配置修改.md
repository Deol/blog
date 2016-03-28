title: "Ubuntu grub 启动管理器配置修改"
date: 2015-05-09 09:50:00
categories: Linux
---

接下来介绍一下安装完 Win 8.1 和 Ubuntu 双系统之后，修改 grub 启动管理器默认启动项的方法。
<!--more-->

通过 grub 选择进入的操作系统，一般是在先装 Windows ，后装 Ubuntu 的情况下。
如果翻转安装顺序，一般我们可以通过 easyBCD 去创建引导项，这里就不展开了。
具体可以参考这里：[使用 easyBCD 引导启动 ubuntu 14.04 ][1]

回到 grub 的问题上，grub 启动的时候会去 Ubuntu 下找 /boot/grub/grub.cfg 文件。
但它实际上是一个由 grub2 的引导程序执行的 bash 类脚本文件，而不是一个真正的配置文件。

所以我们并不直接对其进行修改，而是修改 /etc/default/grub 文件，命令如：

    sudo gedit /etc/default/grub

打开文件之后，我们可以在其中看到这一段配置：

    GRUB_DEFAULT=0
    #GRUB_HIDDEN_TIMEOUT=0
    GRUB_HIDDEN_TIMEOUT_QUIET=true
    GRUB_TIMEOUT=10
    GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
    GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
    GRUB_CMDLINE_LINUX=""

第一行 `GRUB_DEFAULT=0` 的意思是，默认启动启动项列表中的第一项，也就是启动 Ubuntu 。
当然，如果我们对 Windows 的使用频率更高，将数值改为 4，默认启动项变为第 5 项，也就是默认进入 Windows 操作系统。

而 `GRUB_TIMEOUT=10` 这一行代表的是用户等待时间，原本是 10 秒。
如果用户在 10 秒内不进行选择，则默认进入 `GRUB_DEFAULT` 指定的启动项。
个人觉得 10 秒太浪费生命，于是我把它改成了 3 秒。

修改完毕之后保存文件，并执行该命令：

    sudo update-grub
    
该命令会根据刚刚修改的 grub 文件自动生成一个新的 grub.cfg 文件，至此大功告成。

  [1]: http://jingyan.baidu.com/article/1876c852942fea890b13760b.htmlhttp://jingyan.baidu.com/article/1876c852942fea890b13760b.html