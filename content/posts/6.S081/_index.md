+++
author = "Hugo Authors"
date = "2022-06-05"
title = "MIT6.S081 课程学习"
tags = [
    "C",
    "Course",
    "MIT"
    
]
categories = [
    "Operator System"
]
+++

# MIT6.S081课程学习
文章记录了自己学习该门课程的心得，以及一些踩过的坑。同时也是自己第一次写博客的方式进行输出，希望自己以后可以坚持输出。主要内容有以下几部分。
+ [课程简介](#课程简介)
+ [如何学习](#如何学习)
+ [课程解析](#课程解析)
+ [实验](#实验)

## 课程简介
MIT6.S081是一门介绍操作系统的课程，课程主要由[Frans Kaashoek](https://pdos.csail.mit.edu/~kaashoek/) & [Robert Morris](https://pdos.csail.mit.edu/~rtm/)两位教授来进行授课。课程是通过讲解两位教授开发的用于教学的类unix的迷你操作系统xv6，主要是为了理解操作系统内核的工作原理，因此适合想要深度学习和理解操作系统的同学。虽然是一门面向本科生的课程，但我觉得还是有一定难度，如果想要独立完成整个课程包括实验部分，我建议不要将这门课程作为操作系统学习的第一门课程，建议先学习 [CSAPP](https://www.cs.cmu.edu/~213/)，然后再学习这门课程。主要原因有：
1. 这门课程需要有比较好的C语言基础，debug基础
2. CSAPP中的linux基础，汇编，函数调用，页表等知识有助于本门课的理解
3. 个人经验，我其实学了两遍这个课程，第一次直接学习本课程（学了一小部分就放弃了），第二次先完成了CSAPP课程的学习，然后完成了该课程的学习。相信我，有了CSAPP的课程铺垫，大部分实验你都能独立解决，更能游刃有余。

## 如何学习
我觉得这门课程的重点是实验，如果你能够**独立**完成实验，说明你对相应的内容是有足够的了解的，因此，切记、一定、保证**独立**完成实验，宝宝们一定要**独立**完成，:smile:。

学习过程中我遇到比较好的资源

- [X] [视频](https://www.bilibili.com/video/BV19k4y1C7kA?spm_id_from=333.1007.top_right_bar_window_custom_collection.content.click) 
- [X] [翻译](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/) 
- [X] [课程主页](https://pdos.csail.mit.edu/6.828/2020/schedule.html)

翻译是[肖宏辉大神](https://www.zhihu.com/people/xiao-hong-hui-15)对课程每个Lecture内容准确的翻译，搭配视频使用最佳。

课程主页提供了学习课程所需的所有材料，在深入学习这门课之前，需要仔细浏览课程主页，搞清楚课程主页每个部分包含哪些内容。其中以下几个部分是你需要重点关注的。
- [schedule](https://pdos.csail.mit.edu/6.828/2020/schedule.html)：整个课程表，每节课的**video**、**Preparation**、**Assignment**在课程表里都有贴出
- [xv6-book](https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf)：这本书是课程配套的，是课前读物
- [Reference](https://pdos.csail.mit.edu/6.828/2020/reference.html)：一些辅助资料，主要有**unix介绍**、**qemu**、**RISC-V指令集**
- [Labs](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html)：课程的核心部分，后面会详细介绍

接下来说说如何去学习这门课程，也是自己学习的过程，仅限经验参考。总的来说课程的学习分为两大块，课程内容和实验。    

**课程内容**：内容的学习推荐按照【Preparation】->【视频】->【翻译】的顺序。对于每个lecture在课程表里都有对应的预习内容，主要包括[xv6-book](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html)对应章节的内容，以及部分源码的阅读。然后再去看相应的视频，就不会吃力。最后再去看一遍相应的课程内容翻译。可以说，课程内容的学习绝不是简单的一节课80分钟1.5倍速播放的学习:blush:(主要嫌弃语速慢)，前期的Preparation的学习也花费了我许多时间（不要害怕英文，我就六级没过的都不感觉有压力）。  
**实验**：实验切记独立完成，每个实验在课程主页都有相应的介绍，请务必认真阅读。重点每个实验的hints部分，这些hints就是为了引导你完成实验。后续我也会有实验部分的详解，仅供大家参考、交流，如果不自己独立完成实验，而是参考别人的代码，其实收获很小，很多部分自己还是一知半解的。希望大家独立完成。

## 课程解析
主要是对每个lecture的内容的解析，以及一些重点、难点的记录和分析。参考[肖宏辉大神的翻译](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/)和自己的理解整理出来。

> - [Lec01 Introduction and Examples (Robert)](./lectures/lec01)
> - [Lec03 OS Oraganization and System Calls (Frans)](./lectures/lec03)
> - [Lec04 Page tables (Frans)](./lectures/lec04.md)
> - [Lec05 Calling conventions and stack frames RISC-V (TA)](./lectures/lec05)

## 实验
实验部分是自己做实验的思考过程以及踩过的一些坑，详细分析了解决了每一个实验。
> - [Lab0 环境搭建](./labs/lab0/)
> - [Lab1 Utilities](./labs/lab1/)
> - [Lab2 System calls](./labs/lab2/)
> - [Lab3 Page tables](./labs/lab3/)

