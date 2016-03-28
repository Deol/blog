title: "Ubuntu 终端路径相关"
date: 2015-05-08 17:47:00
categories: Linux
---

这阵子折腾 Ubuntu 也是蛮好玩的，在这里分享关于终端的两个小技巧。
<!--more-->

### 在文件夹中，右键在当前位置打开终端

Ubuntu Kylin 的其中一个优点：在文件夹空白处右键时，右键菜单中默认就有「在终端中打开」的选项。
现在只需在终端上输入一行命令，原生 Ubuntu 就也能有这个功能：

    sudo apt-get install nautilus-open-terminal

说到底只是装上了 nautilus-open-terminal 而已，所以我们也能在 Ubuntu 直接搜索找到软件并安装。

之后注销再登录就可以了。

---

### 终端路径只显示当前目录

完成上面一步之后，我们会发现这样打开的终端路径可能会很长，极为不便也不美观。
这时候我们可以通过命令：
    
    sudo gedit ~/.bashrc

找到 .bashrc 文件中的这一部分：

    if [ "$color_prompt" = yes ]; then
        PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
    else
        PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w\$ '

将这段代码中的两个小写`w`改成大写`W`即可。