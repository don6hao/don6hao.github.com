---
layout: post
title: "Libeio--libeio流程分析"
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

want_poll函数是worker线程调用，done_poll是主线程调用。libeio要user通过这两个函数实现worker线程和主线程之间的通信。


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

eio_finish->req->finish函数（eio_nop中把res_cb赋值给req->finish)

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
        /*
         * 创建管道
         * 在worker线程完成IO请求，通知主线程的机制是需要使用者自定义的
         * 这里我们使用pipe(一种常用的线程通知机制）作为通信机制
         */
        if (pipe(respipe)) abort ();
        printf ("eio_init ()\n");

        //eio_init初始化回调函数和res_queue/req_queue等相关数据
        if (eio_init (want_poll, done_poll)) 
            abort ();

        do{
            /*
             * eio_nop : 把参数封装成request(eio_req)并eio_submit操作
             *
             * eio_submit会进行以下操作:
             * 1. 把request放入到req_queue队列中(reqq_push操作)
             * 2. 告知worker线程有请求到达(cond_signal操作)。
             * 3. eio_submit会启动一个work线程

             * 启动的work线程会进行以下操作：
             * 1. reqq_shift从req_queue取数据。
             * 2. cond_wait等待cond_signal的信息。
             * 3. 把req放入到res_queue队列中，若want_poll_cb非空，执行want_poll_cb(want_poll_cb指向want_poll)。
             */

            eio_nop(0, res_cb, "nop");
            event_loop();
        }while (0);

        return 0;
    }


eio_submit函数
---
    ETP_API_DECL void
    etp_submit (ETP_REQ *req)
    {
        req->pri -= ETP_PRI_MIN;

        if (ecb_expect_false (req->pri < ETP_PRI_MIN - ETP_PRI_MIN)) req->pri = ETP_PRI_MIN - ETP_PRI_MIN;
        if (ecb_expect_false (req->pri > ETP_PRI_MAX - ETP_PRI_MIN)) req->pri = ETP_PRI_MAX - ETP_PRI_MIN;

        if (ecb_expect_false (req->type == ETP_TYPE_GROUP))
        {
            /* group request */
            /* I hope this is worth it :/ */
            X_LOCK (reqlock);
            ++nreqs;
            X_UNLOCK (reqlock);

            X_LOCK (reslock);

            ++npending;

            if (!reqq_push (&res_queue, req) && want_poll_cb)
            want_poll_cb ();

            X_UNLOCK (reslock);
        }
        else
        {
            X_LOCK (reqlock);
            ++nreqs;
            ++nready;
            /* 把request放入到req_queue队列中(reqq_push操作) */
            reqq_push (&req_queue, req);
            /* 告知worker线程有请求到达(cond_signal操作) */
            X_COND_SIGNAL (reqwait);
            X_UNLOCK (reqlock);

            /* 
             * 启动的work线程会进行以下操作：
             * 1. reqq_shift从req_queue取数据。
             * 2. cond_wait等待cond_signal的信息。
             * 3. 把req放入到res_queue队列中，若want_poll_cb非空，执行want_poll_cb(want_poll_cb指向want_poll)。
             */
            etp_maybe_start_thread ();
        }
    }

worker线程
---
work线程调用etp_proc函数。[libeio线程](http://don6hao.github.io/blog/2015/01/12/libeio_data.html)

    static void ecb_cold
    etp_start_thread (void)
    {
        etp_worker *wrk = calloc (1, sizeof (etp_worker));

        /*TODO*/
        assert (("unable to allocate worker thread data", wrk));

        X_LOCK (wrklock);

        /* 调用etp_proc函数 */
        if (xthread_create (&wrk->tid, etp_proc, (void *)wrk))
        {
            wrk->prev = &wrk_first;
            wrk->next = wrk_first.next;
            wrk_first.next->prev = wrk;
            wrk_first.next = wrk;
            ++started;
        }
        else
            free (wrk);

        X_UNLOCK (wrklock);
    }



etp_proc函数
---

若req_queue有数据时:
1. 调用ETP_EXECUTE->eio_execute(eio_execute最底层的处理函数,根据请求的类型调用相应函数操作),把结果返回给req.
2. 把req数据push到res_queue中，然后调用want_poll_cb回调函数(通知主线程有数据可读)

若req_queue无数据时:
libeio只允许max_idle个线程处于空闲等待X_COND_WAIT，从第max_idle+1个线程开始超时等待（若超时就线程退出）

    #define X_THREAD_PROC(name) static void *name (void *thr_arg)
    X_THREAD_PROC (etp_proc)
    {
        ETP_REQ *req;
        struct timespec ts;
        etp_worker *self = (etp_worker *)thr_arg;

        etp_proc_init ();

        /* try to distribute timeouts somewhat evenly */
        ts.tv_nsec = ((unsigned long)self & 1023UL) * (1000000000UL / 1024UL);

        for (;;)
        {
            ts.tv_sec = 0;

            X_LOCK (reqlock);

            for (;;)
            {
                req = reqq_shift (&req_queue);

                /* req_queue有数据就跳出循环 */
                if (req)
                    break;

                if (ts.tv_sec == 1) /* no request, but timeout detected, let's quit */
                {
                    /* 超时就线程退出 */
                    X_UNLOCK (reqlock);
                    X_LOCK (wrklock);
                    --started;
                    X_UNLOCK (wrklock);
                    goto quit;
                }

                ++idle;

                /* libeio只允许max_idle个线程处于空闲等待X_COND_WAIT
                 * 从第max_idle+1个线程开始，进行超时等待判断（若超时就线程退出）
                 */
                if (idle <= max_idle)
                    /* we are allowed to idle, so do so without any timeout */
                    X_COND_WAIT (reqwait, reqlock);
                else
                {
                    /* initialise timeout once */
                    if (!ts.tv_sec)
                        ts.tv_sec = time (0) + idle_timeout;

                    if (X_COND_TIMEDWAIT (reqwait, reqlock, ts) == ETIMEDOUT)
                        ts.tv_sec = 1; /* assuming this is not a value computed above.,.. */
                }

                --idle;
            }

            --nready;

            X_UNLOCK (reqlock);

            /* 收到线程退出的request */
            if (req->type == ETP_TYPE_QUIT)
                goto quit;

            ETP_EXECUTE (self, req);

            X_LOCK (reslock);

            ++npending;

            if (!reqq_push (&res_queue, req) && want_poll_cb)
                want_poll_cb ();

            etp_worker_clear (self);

            X_UNLOCK (reslock);
        }

    quit:
        free (req);

        X_LOCK (wrklock);
        /* 从线程池中移除 */
        etp_worker_free (self);
        X_UNLOCK (wrklock);

        return 0;
    }


eio_poll->etp_poll
---
主要是完成res_queue中就绪的eio_req对象的处理。ETP_FINISH宏中会调用eio_req中绑定的回调函数(上面测试代码中绑定的函数就是res_cb函数).

    #define EIO_FINISH(req)  ((req)->finish) && !EIO_CANCELLED (req) ? (req)->finish (req) : 0

    ETP_API_DECL int
    etp_poll (void)
    {
        unsigned int maxreqs;
        unsigned int maxtime;
        struct timeval tv_start, tv_now;

        X_LOCK (reslock);
        maxreqs = max_poll_reqs;
        maxtime = max_poll_time;
        X_UNLOCK (reslock);

        if (maxtime)
            gettimeofday (&tv_start, 0);

        for (;;)
        {
            ETP_REQ *req;

            etp_maybe_start_thread ();

            X_LOCK (reslock);
            req = reqq_shift (&res_queue);

            /*
             * 取出数据然后调用done_poll_cb回调函数读取线程间的信息
             */
            if (req)
            {
                --npending;
                if (!res_queue.size && done_poll_cb)
                    done_poll_cb ();
            }

            X_UNLOCK (reslock);

            if (!req)
                return 0;

            X_LOCK (reqlock);
            --nreqs;
            X_UNLOCK (reqlock);

            if (ecb_expect_false (req->type == ETP_TYPE_GROUP && req->size))
            {
                req->int1 = 1; /* mark request as delayed */
                continue;
            }
            else
            {
                /*
                 * 上面测试代码eio_nop中把res_cb赋值给req->finish（ETP_FINISH->eio_finish->EIO_FINISH)
                 */
                int res = ETP_FINISH (req);
                if (ecb_expect_false (res))
                    return res;
            }

            if (ecb_expect_false (maxreqs && !--maxreqs))
                break;

            if (maxtime)
            {
                gettimeofday (&tv_now, 0);

                if (etp_tvdiff (&tv_start, &tv_now) >= maxtime)
                break;
            }
        }

        errno = EAGAIN;
        return -1;
    }


