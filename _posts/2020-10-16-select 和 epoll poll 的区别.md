---
layout: post
title: 'select和epoll的区别'
date: 2020-10-16
author: Warmingzzz
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: 网络模型

---

### IO多路复用之select、poll、epoll详解

IO多路复用是指内核一旦发现进程指定一个或者多个IO条件准备读取，他就通知该线程。IO多路复用适合如下场景：

​	1、当客户处理多个描述符时（一般是交互式输入和网络套接口），必须使用IO复用

​	2、当一个客户同时处理多个套接口时，而这种情况时可能的，但很少出现

​	3、当一个TCP服务器既要处理监听套接口，又要处理医连接套接口，一般也要用到IO复用

​	4、如果一个服务器既要处理TCP、UDP，一般要IO复用

​	5、如果一个服务器要处理多个服务或多个协议，一般要使用IO复用



与多进程多线程技术相比，**IO多路复用的最大优势时系统开销小，系统不必创建进程/线程，也不必维护这些进程/线程**，从而大大减小了系统的开销。

目前支持IO多路复用的系统调用有select、pselect、poll、epoll，IO多路复用就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读or写就绪），能够通知程序进行相应的读写操作。但select、pselect、poll、epoll本质都是同步IO，因为他们都需要在读写事件就绪或自己负责读写，也就是这个读写过程时堵塞的，而异步IO则无需自己负责进行读写，异步IO的实现会负责把数据从内核拷贝到用户空间。



### 1、select、poll、epoll简介

epoll跟select都能提供多路复用IO的解决方案。epoll是LINUX特有的，而select是posix所规定的，一般操作系统均有实现。

1.1 select

select目前在所有的平台上都支持，其良好的跨平台性也是他的一个优点。select的一个缺点在于单个进程能够监视的文件描述符的数量存在最大限制，在linux上一般为1024，可以通过修改宏定义甚至重新编译内核的方式提升这个限制，但是这样也会导致效率的降低。

1.select最大的缺陷是单个进程打开的FD是有一定限制的，默认是1024(32位机默认1024个，64位机默认是2048个)

2.对socekt进行扫描是线性扫描，采用轮询的方法，效率较低。



### 1.2 poll

 他没有最大连接数的限制，原因是它基于链表来存储的，但是同样有一个缺点：

1.大量的fd的数组被整体复制与用户态和内核地址之间，可能是无意义的

2.poll还有一个特点是水平触发，如果报告了fd后，没有被处理，那么下次poll时会再次报告改fd。

### 1.3 epoll

epoll更加灵活，没有描述符的限制。**epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个时间表中，这样在用户空间和内核空间之间只需要copy一次。**

epoll的优点：

1、没有最大并发连接的限制，能打开的FD的上限远大于1024

2、效率提升，不是轮训的方式，不会随着FD数目的多增加效率下降。只有活跃可用的FD才会调用callback函数，和总数无关。效率高

3、内存拷贝，利用mmap（）文件映射内存加速与内核空间的消息传递，**即epoll使用mmap减少复制开销。**



## 2、select、poll、epoll区别

#### 1、支持一个进程所能打开的连接数

| 方法   | 区别                                                         |
| ------ | :----------------------------------------------------------- |
| select | 单个进程有最大连接数（32位是32X32，64位时32X64），可以修改最大连接数，性能可以会有影响 |
| poll   | poll本质和select没有区别，没有最大连接数限制，因为他是用链表存 |
| epoll  | 虽然连接数有上限，但是很大，1G内存的机器可以打开10W个连接    |

#### 2.FD剧增后带来的IO效率问题

| 方法   | 区别                                                         |
| ------ | ------------------------------------------------------------ |
| select | 因为每次都会线性遍历，所以FD增大效率会线性下降               |
| poll   | 同上                                                         |
| epoll  | epoll内核中实现时每个fd上的callback函数实现的，只有活跃的socket才会主动调用callback，所以在活跃socket较少的情况下，使用epoll没有前面两者的线性下降的性能问题 |

#### 3、消息传递方式

| 方法   | 区别                                             |
| ------ | ------------------------------------------------ |
| select | 内核需要将信息传递到用户空间，都需要内核拷贝动作 |
| poll   | 同上                                             |
| epoll  | epoll通过内核和用户空间共享一块内存实现          |





### 总结

1、表面上select性能最好，但是连接数少并且都十分活跃的情况下，select和poll的性能可能比epoll好，毕竟epoll的通知机制需要很多函数回调。

2、select低效是应为他每次都需要轮询。