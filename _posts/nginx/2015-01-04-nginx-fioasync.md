---
layout: post
title: "Nginx--FIOASYNC"
description: "fioasync nginx"
category: Nginx
tags: []
---

>ngx_spawn_process函数中设置信号驱动异步I/O标志,它决定是否收取针对本套接口的异步I/O
信号(SIGIO).

    {
        on = 1;
        if (ioctl(ngx_processes[s].channel[0], FIOASYNC, &on) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "ioctl(FIOASYNC) failed while spawning \"%s\"", name);
            ngx_close_channel(ngx_processes[s].channel, cycle->log);
            return NGX_INVALID_PID;
        }

        if (fcntl(ngx_processes[s].channel[0], F_SETOWN, ngx_pid) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "fcntl(F_SETOWN) failed while spawning \"%s\"", name);
            ngx_close_channel(ngx_processes[s].channel, cycle->log);
            return NGX_INVALID_PID;
        }
    }

FIOASYNC
---
FIOASYNC Enables a simple form of asynchronous I/O notification. 
This command causes the kernel to send SIGIO signal to a process or a process group when I/O is possible. 
Only sockets, ttys, and pseudo-ttys implement this functionality.


WHEN DO WE USE FIOASYNC
---
Let's imagine a process that executes a long computational loop at low priority but needs to process incoming data as soon as possible. 
If this process is responding to new observations available from some sort of data acquisition peripheral, it would like to know immediately when new data is available. 
This application could be written to call poll regularly to check for data, but, for many situations, there is a better way. 
**By enabling asynchronous notification, this application can receive a signal
whenever data becomes available** and need not concern itself with polling.

HOWTO USE FIOASYNC
---
User programs have to execute **two steps** to enable asynchronous notification from an input file. 
First, they specify a process as the "owner" of the file. 
When a process invokes the *F_SETOWN* command using the fcntl system call, the process ID of the owner process is saved in filp->f_owner for later use. 
This step is necessary for the kernel to know just whom to notify.

    fcntl(fd, F_SETOWN, process_id)

In order to actually enable asynchronous notification, the user programs must
set the **FASYNC flag** in the device by means of the F_SETFL fcntl command as
blow:

    oflags = fcntl(STDIN_FILENO, F_GETFL);
    fcntl(STDIN_FILENO, F_SETFL, oflags | FASYNC);

or call ioctl function as blow:

    int on = 1;ioctl(fd, FIOASYNC, &on)

After these two calls have been executed, the input file can request delivery of a SIGIO signal whenever new data arrives. 
The signal is sent to the process (or process group, if the value is negative) stored in filp->f_owner.
