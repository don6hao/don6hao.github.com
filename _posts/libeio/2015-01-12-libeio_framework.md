---
layout: post
title: "Libeio--libeio流程"
description: "libeio asynchronous framework"
category: Libeio
tags: []
---


![libeio_framework_1.png](./../../../../../../pic/libeio_framework_1.png) 

测试代码
---

    #include <stdio.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <poll.h>
    #include <string.h>
    #include <assert.h>
    #include <fcntl.h>
    #include <sys/types.h>
    #include <sys/stat.h>

    #include "eio.h"

    int respipe [2];

    /* 
     * want_poll回调函数，赋值给want_poll_cb
     * worker线程通知主线程的机制是通过向 pipe[1]写一个 byte
     * 数据(event_loop->poll)
     */

    void want_poll (void)
    {
        char dummy;
        printf ("want_poll ()\n");
        write (respipe [1], &dummy, 1);
    }

    //done_poll回调函数,赋值给done_poll_cb
    /* 
     * done_poll回调函数,赋值给done_poll_cb
     * done_poll 从 pipe[0]读出一个 byte 数据，该 IO 操作完成
     */


    void done_poll (void)
    {
        char dummy;
        printf ("done_poll ()\n");
        read (respipe [0], &dummy, 1);
    }

    //事件循环

    void event_loop (void)
    {
        // an event loop. yeah.
        struct pollfd pfd;
        pfd.fd     = respipe [0];
        pfd.events = POLLIN;

        printf ("\nentering event loop\n");

        // eio_nreqs返回当前正在处理的请求数量。main函数只加入一个request请求eio_nreqs()等于1.

        while (eio_nreqs()){
            /* 
             * 等待worker线程的通知,当pipe[0]可读时，就调用eio_poll
             */
            poll(&pfd, 1, -1);

            /*
             * eio_poll->etp_poll完成：
             * 1. 调用reqq_shift从res_queue取出数据(eio_req)。
             * 2. 调用don_poll_cb回调函数。
             * 3. ETP_FINISH->eio_finish->req->finish函数(回调函数，本例中相当于res_cb函数）
             */

            printf("eio_poll() = %d\n", eio_poll());
        }

        printf ("leaving event loop\n");
    }

    //eio_finish->req->finish函数（eio_nop中把res_cb赋值给req->finish)

    int res_cb (eio_req *req)
    {
        printf ("res_cb(%d|%s) = %d\n", req->type, req->data ? req->data : "?", EIO_RESULT (req));

        if (req->result < 0)
            abort ();

        return 0;
    }

    int main (void)
    {
        printf ("pipe ()\n");
        //建立管道
        if (pipe(respipe)) abort ();
        printf ("eio_init ()\n");

        //eio_init初始化回调函数和res_queue/req_queue等相关数据
        if (eio_init (want_poll, done_poll)) 
            abort ();

        do{
            /*
             * eio_nop 
             * 1. 把参数封装成request(eio_req)并eio_submit操作
             * 2. 把request放入到req_queue队列中(reqq_push操作)
             * 3. 告知worker线程有请求到达(cond_signal操作)。

             * eio_submit会启动一个work线程：
             * 1. reqq_shift从req_queue取数据。
             * 2. cond_wait等待cond_signal的信息。
             * 3. 把req放入到res_queue队列中，若want_poll_cb非空，执行want_poll_cb(want_poll_cb指向want_poll)。
             */

            eio_nop(0, res_cb, "nop");
            event_loop();
        }while (0);

        return 0;
    }

