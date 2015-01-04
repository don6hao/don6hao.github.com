---
layout: post
title: "TCP/IP-socket layer"
description: "socket fucntion implementation"
category: TCP_IP.Architecture,.Design.and.Implementation.in.Linux
tags: []
---

BSD socket
---
The BSD socket is a framework to the different families of socket that Linux supports. 
The BSD socket concept is very similar to the VFS (virtual file system) layer.
This way different protocol families are supported by Linux, and their services are accessable to the user using a common socket interface.

###VFS
VFS is a framework that provides a common interface to various different file systems/pipe/devices/sockets to the user without user knowing how things are organized inside the kernel.
![Figure_3.1](./../../../../../../pic/Figure_3.1.png) 


socket()
---
用户调用socket函数时会调用内核层的sys_socket函数.sys_socket函数通过socket函数传递来的protocol,family
,type三个参数来创建对应的(TCP/UDP/RAW等)协议栈。

socket()函数在内核中执行流程：

>socket()->sys_socketcall->sys_socket()->sock_create()

    asmlinkage long sys_socket(int family, int type, int protocol)
    {
        int retval;
        struct socket *sock;
        /*
         * sock_create会调用__sock_create函数
         * 若family为PF_INET,就会调用inet_create函数来初始化socket结构体
         */
        retval = sock_create(family, type, protocol, &sock);
        if (retval < 0)
            goto out;

        /*
         * 把struct socket sock放入到VFS中，返回fd-套接字
         * 图Figure_3.2
         */
        retval = sock_map_fd(sock);
        if (retval < 0)
            goto out_release;

    out:
        /* It may be already another descriptor 8) Not kernel problem. */
        return retval;

    out_release:
        sock_release(sock);
        return retval;
    }


![Figure_3.2](./../../../../../../pic/Figure_3.2.png) 
