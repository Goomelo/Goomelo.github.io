---
title: build AOSP on riscv (尝试参与AOSP在riscv上移植)
date: 2021-12-22 01:36:48
tags:
---
首先感谢软件所PLCT实验室汪辰老师的文章，带我入门了aosp的一些基本概念并且跟着老师的教程完成了环境的搭建。

本来是想用英文记录下整个参与过程，但是英语水平有限。

机缘巧合是之前稍稍入坑了安卓开发，稍微了解了一下基于安卓的构建之后，发现了aosp官方一直在维护的仓库，又了解到了国内有一批
人（没错就是平头哥）做了aosp在riscv上的移植，而我又十分看好riscv的未来，也非常欣赏riscv的设计，因此想到了有没有机会来参与一下
aosp在riscv上的移植。

本文大概的框架非常简单，因为主要是记录参与AOSP移植的一些过程，记录一些踩过的坑，因此首先会介绍AOSP的基本概念，
包括AOSP的前置知识包括仓库里的版本管理等知识，之后会介绍我在使用riscv工具链编译Linux的一些小小经验（当然工具链的验证
等工作都由PLCT的前辈完成，本人只是做一些拾人牙慧的操作），最后是基于qemu上riscv-64位内核编译Linux以及编译riscv-64 AOSP的经验。

# 关于AOSP你需要知道什么
AOSP是Android Open Source Project的简称，即安卓开源项目，实际上就是安卓开源系统啦，我们所熟知的MIUI，H2OS等等用在安卓手机伤的操作系统，
大都是基于AOSP进行开发的。官方仓库可以在这里获取：https://source.android.com/?hl=zh-cn

众所周知安卓系统可以看作是运行在移动设备上的Linux系统，虽然Android+Arm的生态环境几乎已经形成类似WinTel一般牢不可破的联盟，但笔者依然看好Riscv
在移动市场的未来前景，因此看到了aosp on riscv的相关项目便义无反顾入了坑。

## 版本管理
版本管理在21世纪的今天已经不是陌生的概念，而面对AOSP这样庞大的时逾15年的大型开源项目就更为重要。

安卓的每一个开发版本有一个叫做platform的概念，基于此Google提出了三个与版本管理相关的概念：
* 每个Platform有一个唯一用于标示的版本号Version，这当然非常易于理解。版本号采用点分格式，即X.Y.Z的格式，
一般简写为X或X.Y，从1.0开始一直到目前发现的11。
  
* Google也是个很有意思的公司，因此不同的发行版也会有一个独立的Codename，有点类似macos起昵称的感觉，还是
蛮可爱的设定。当然Codename和Version是一对多的概念，用过macos的小伙伴应该可以理解。
  
* 与Version对应的概念是API level，顾名思义就是该Version提供给上层应用的编程接口，因此API level对应的
Version是唯一的。
  
与之相关更关，在Platform层面更细的概念就是tag和build了，tag与git里的tag类似，build则是google的构建版本，
在此不再赘述。（本来还想写一点关于构建工具的文章，但是感觉完全没有个人产出了，有点生搬硬套的感觉，遂放弃）。

### 内核版本管理
如果有build过Linux内核经验的同学，应该对这个概念并不陌生，本文主要介绍的是AOSP内核，因其使用的Linux内核是在原生Linux内核上，
加了一些安卓系统所需的特性（根据wangchen老师的说法是短期内还没有被Linux接纳，以至于未被合并到Linux upstream的补丁程序）。

总而言之，随着Linux内核的不断演进发展以及Android内核自身的演进，对于AOSP内核管理最重要的事情就是管理好Linux内核与AOSP内核之间的关系，
包括如何利用上Linux upstream上的新功能和特性，兼容之后之前的特性等等。

#### Linux内核
从 2003 年 12 月往后，也就是 从 2.6内核版本发布往后，内核开发者社区从之前维护单独的一个开发分支和另一个稳定分支的模型迁移到只维护一个 “稳定” 分支模型。
在此模式下，每 2-3 个月就会发布一次新版本，出现这种变化的原因在于：2.6 版本内核之前的版本周期非常长（将近 3 年），且同时维护两个不同的代码库难度太高。

内核版本的编号从 2.6.x 开始，其中 x 是一个数字，会在每次发布新版本时递增（除了表示此版本比上一内核版本更新之外，该数字的值不具有任何意义）。内核版本发展到现在已经进入 5 系列（最新的是 5.8.x）。
在众多内核版本中，分为 mainline 即主线版本，stable 稳定版本和 longterm 长期维护版本。其中 mainline 版本由 Linus 本人维护，每两到三个月会做一次升级，而且是两位数字版本号，
不涉及第三位数字，譬如 5.1、5.2、5.3 ......。stable 版本是针对 mainline 版本你的小版本更新，譬如对应 5.1 的会有 5.1.1、5.1.2、5.1.3 ......。某一些 stable 版本会被指定为 longterm 版本（也称为 LTS），
从而拥有更长的维护周期，直白了说就是其第三位数字会会变得很大。
（这段偷懒了直接复制了）

#### AOSP内核
AOSP内核主要了解的概念是"Android platform release"概念，Google基本保持一年一更新的概念。
其中Launch Kernel的概念即AOSP内核用于发布在新设备上的版本，Feature Kernel则是为了支持Android某个Platform新特性的类似布丁的更新。
