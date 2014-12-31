---
layout: post
title: "socket layer"
description: "howto the function of socket() implement"
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
