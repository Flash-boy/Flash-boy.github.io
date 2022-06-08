+++
author = "Hugo Authors"
date = "2022-06-05"
title = "Lab0 环境搭建"
tags = [
    "C",
    "Course",
    "MIT"
    
]
categories = [
    "Operator System"
]
+++
# 环境搭建
## 写在前面
要完成课程实验，希望你可以主动去学习一些工具的使用，将有助于高效完成整个实验，这些工具也是作为一个linux系统开发人员必备的知识，我自己常用的工具主要有以下几个：
- [tmux](https://www.man7.org/linux/man-pages/man1/tmux.1.html)：终端复用工具
- [vim](https://missing.csail.mit.edu/2020/editors/)：代码编辑器
- [git](https://missing.csail.mit.edu/2020/version-control/)：版本控制工具
- [gdb](https://www.sourceware.org/gdb/)：代码调试器

**tmux**：做整个实验，我们必然需要打开多个文件，查看多个xv6内核的源码，tmux可以帮住我们同时打开多个窗口，并且在多个窗口来回切换。相信我真的特别好用。:wink:   
**vim**：linux自带的代码编辑器，不必多说，喜欢的喜欢不得了，不用的嫌麻烦，我觉得还是很有必要去学习的。vim是学习成本比较高的，学习路线比较陡峭，学到手收益也是很大的。   
**git**：每个开发人员都应该掌握的代码管理工具，学习git不仅仅是学习命令的使用，更应该学习它的底层实验原理，这样会更加理解每个命令的实际作用。  
**gdb**：linux下的代码调试工具，是真的好用，也是该课程的调试工具。

可能以上工具觉得有点杂，但是有一门由MIT助教发起的课程，叫做**计算机教育中缺失的一课**。该课程不长10个小时左右，就介绍了上述的各种工具，强烈建议可以去学习一下，对于工作开发十分有用，这里是该课的[课程主页](https://missing.csail.mit.edu/)和视频资源，[B站](https://www.bilibili.com/video/BV1x7411H7wa?spm_id_from=333.1007.top_right_bar_window_custom_collection.content.click)，[youtube](https://www.youtube.com/channel/UCuXy5tCgEninup9cGplbiFw)。

通过**计算机教育中缺失的一课**我还学习到了一个很重要的工具**dotfiles**，如何管理你的每个工具的配置文件，当你在切换工作机器或者电脑时，能很方便的配置你的计算机，定制化你的配置，包括shell、vim、tmux、git等。这门课的主讲人之一[Anish Athalye](https://github.com/anishathalye)开发了一个[dotbot](https://github.com/anishathalye/dotbot),可以很方便的管理你的dotfiles。我参考他的dotfiles编写了自己的[dotfiles](https://github.com/Flash-boy/dotfiles),有需要的自取^_^。

当然，也希望你目前能够翻墙，因为这有助于你在linux下下载某些软件包时不会因为网络原因无法成功。懂的都懂 :smirk:

## [官方指导配置环境](https://pdos.csail.mit.edu/6.828/2020/tools.html)
官方给出了不同机器的配置方式，这里主要介绍在window下利用虚拟机进行配置。推荐采用ubuntu20.04系统，因为没什么坑:joy:。推荐使用[Virtual Box](https://www.virtualbox.org/)安装[ubuntu20.04](https://ubuntu.com/#download)。 

使用官方给的命令即可初步配置环境
```shell
sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu 

sudo apt-get remove qemu-system-misc

sudo apt-get install qemu-system-misc=1:4.2-3ubuntu6 

```
接着安装官方指引确认以下两个软件已经安装

```shell
~ > riscv64-unknown-elf-gcc --version
riscv64-unknown-elf-gcc () 9.3.0
...

# 可以看到输出的版本并没有官方指导书上一致，没有关系
# 如果出现了 Command 'riscv64-unknown-elf-gcc' not found 字样，说明没有安装该工具，运行下面命令即可
# sudo apt install gcc-riscv64-unknown-elf
```
```shell
~ > qemu-system-riscv64 --version
QEMU emulator version 4.2.1 (Debian 1:4.2-3ubuntu6.21)
# 同样的也可以看到版本和官方提供的不一致，没关系
```
接着就可以下载官方提供的实验代码
```shell
~/Desktop > git clone git://g.csail.mit.edu/xv6-labs-2020
Cloning into 'xv6-labs-2020'...
...
```
切换到对应的代码目录下
```shell
~/Desktop > cd xv6-labs-2020 
~/Desktop/xv6-labs-2020 > ls 
# 可以发现该目录下并没有代码，这是因为代码通过git管理
# 不同实验需要切换到不同分支，首先通过以下命令切换到
# lab1 Utilities所在分支
~/Desktop/xv6-labs-2020 > git checkout util    
```
然后执行以下命令就可以编译xv6
```shell
~/Desktop/xv6-labs-2020 > make qemu   
```
如果成功可以看到如下图片
![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab0-1.png)

现在虽然能够编译运行xv6，但按照官方的指导仍然不能使用gdb调试内核，所以我们需要利用gdb-multiarch启动gdb调试
首先运行以下命令
```shell
~/Desktop/xv6-labs-2020 > echo "add-auto-load-safe-path XV6_LAB_PATH/.gdbinit" >> ~/.gdbinit 
# 这里XV6_LAB_PATH是指你下载的实验代码所在目录例如我的就是 ~/Desktop/xv6-labs-2020，替换成你的实验代码所在目录即可
```

然后开启两个窗口，分别运行以下两命令
```
~/Desktop/xv6-labs-2020 > make CPUS=1 qemu-gdb
~/Desktop/xv6-labs-2020 > gdb-multiarch
```
可以看到成功的效果如图所示
![](https://raw.githubusercontent.com/Flash-boy/PicGo/master/6.S081/lab0-2.png)

至此完成了所需的实验环境安装，当然你也可以按照官方指导书自己去编译相关的工具。
## docker配置环境 
利用docker容器可以很方便的完成环境搭建，具体参考[这里](https://www.bilibili.com/video/BV1Qi4y1o7tN/?spm_id_from=333.788)。


