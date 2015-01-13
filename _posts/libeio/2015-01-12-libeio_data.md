---
layout: post
title: "Libeio--libeio初始化，REQ/RES队列，锁，线程池"
description: "libeio asynchronous"
category: Libeio 
tags: []
---

libeio 初始化
---
    int eio_init (void (*want_poll)(void), void (*done_poll)(void))

eio_init函数：初始化libeio库。成功返回0，失败返回-1并且设置合适的errno值。
1. 初始化libeio中的一些全局结构，比如：req_queue，res_queue，以及各种互斥量等。
2. 保存外界传入的两个回调函数：want_poll和done_poll。这两个函数都是边缘触发函数

    etp_init (void (*want_poll)(void), void (*done_poll)(void))
    {

        //初始化三个互斥量wrklock, reslock, reqlock和一个条件变量reqwait

        X_MUTEX_CREATE (wrklock);
        X_MUTEX_CREATE (reslock);
        X_MUTEX_CREATE (reqlock);
        X_COND_CREATE  (reqwait);

        /*
         *初始化两个队列：
         * 1.eio_submit把request放入req_queue
         * 2.eio_poll从res_queue中取出数据
         */

        reqq_init (&req_queue);
        reqq_init (&res_queue);

        //work线程链表

        wrk_first.next =
        wrk_first.prev = &wrk_first;

        started  = 0;
        idle     = 0;
        nreqs    = 0;
        nready   = 0;
        npending = 0;

        //设置回调函数

        want_poll_cb = want_poll;
        done_poll_cb = done_poll;

        return 0;
    }


REQ队列和RES队列
---

etb_reqq结构体

    typedef struct eio_req    eio_req;
    #define ETP_REQ eio_req

    /*
    * a somewhat faster data structure might be nice, but
    * with 8 priorities this actually needs <20 insns
    * per shift, the most expensive operation.
    */
    typedef struct {
        ETP_REQ *qs[ETP_NUM_PRI], *qe[ETP_NUM_PRI]; /* qstart, qend */
        int size;
    } etp_reqq;


定义两个etp_reqq全局变量

    static etp_reqq req_queue;
    static etp_reqq res_queue;

初始化队列

    static void ecb_noinline ecb_cold
    reqq_init (etp_reqq *q)
    {
        int pri;

        /*
         * etp_reqq包含ETP_NUM_PRT个指向ETP_REQ的指针数组
         * 每个数组成员的头指针和尾指针指向0 
         */
        for (pri = 0; pri < ETP_NUM_PRI; ++pri)
            q->qs[pri] = q->qe[pri] = 0;

        q->size = 0;
    }

把request放入到队列中

    static int ecb_noinline
    reqq_push (etp_reqq *q, ETP_REQ *req)
    {
        int pri = req->pri;
        req->next = 0;

        /*
         * 若qe[pri]已有数据则该更新队列的尾指针指向该request
         * 若无数据则队列头指针和尾指针都指向该request
         */
        if (q->qe[pri])
        {
            q->qe[pri]->next = req;
            q->qe[pri] = req;
        }
        else
            q->qe[pri] = q->qs[pri] = req;

        return q->size++;
    }


把request从队列中取出

    static ETP_REQ * ecb_noinline
    reqq_shift (etp_reqq *q)
    {
        int pri;

        if (!q->size)
            return 0;

        --q->size;

        for (pri = ETP_NUM_PRI; pri--; )
        {
            ETP_REQ *req = q->qs[pri];

            if (req)
            {
                /* 队列数据取完，头和尾指针指向0 */
                if (!(q->qs[pri] = (ETP_REQ *)req->next))
                    q->qe[pri] = 0;

                return req;
            }
        }

        /* 无数据可取，异常退出 */
        abort ();
    }

libeio中的锁
---

    #include <pthread.h>
    #define sigset_t int
    #define sigfillset(a)
    #define pthread_sigmask(a,b,c)
    #define sigaddset(a,b)
    #define sigemptyset(s)

    typedef pthread_mutex_t xmutex_t;
    #define X_MUTEX_INIT           PTHREAD_MUTEX_INITIALIZER
    #define X_MUTEX_CREATE(mutex)  pthread_mutex_init (&(mutex), 0)
    #define X_LOCK(mutex)          pthread_mutex_lock (&(mutex))
    #define X_UNLOCK(mutex)        pthread_mutex_unlock (&(mutex))

    typedef pthread_cond_t xcond_t;
    #define X_COND_INIT                     PTHREAD_COND_INITIALIZER
    #define X_COND_CREATE(cond)		pthread_cond_init (&(cond), 0)
    #define X_COND_SIGNAL(cond)             pthread_cond_signal (&(cond))
    #define X_COND_WAIT(cond,mutex)         pthread_cond_wait (&(cond), &(mutex))
    #define X_COND_TIMEDWAIT(cond,mutex,to) pthread_cond_timedwait (&(cond), &(mutex), &(to))


在eio_submit中的部分代码：
1.reqlock加锁进行数据更新和reqq_push操作然后reqlock解锁
2.发送条件锁的信号给worker线程

    X_LOCK (reqlock);
    ++nreqs;
    ++nready;
    reqq_push (&req_queue, req);
    X_COND_SIGNAL (reqwait);
    X_UNLOCK (reqlock);


etp_poll中的部分代码：
1.reqlock加锁进行数据更新和reqq_shift操作然后reqlock解锁

    X_LOCK (reslock);
    req = reqq_shift (&res_queue);
    if (req)
    {
        --npending;
        if (!res_queue.size && done_poll_cb)
        done_poll_cb ();
    }
    X_UNLOCK (reslock);


worker线程中使用X_COND_WAIT或X_COND_TIMEDWAIT来接受条件锁信号

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


work线程池
---

线程池初始化：
通过calloc分配work线程资源并挂在wrk_first双向链表上

    static etp_worker wrk_first; /* NOT etp */

    static void ecb_cold
    etp_start_thread (void)
    {
        etp_worker *wrk = calloc (1, sizeof (etp_worker));

        /*TODO*/
        assert (("unable to allocate worker thread data", wrk));

        X_LOCK (wrklock);

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

从链表上移除

    static void ecb_cold
    etp_worker_free (etp_worker *wrk)
    {
        free (wrk->tmpbuf.ptr);

        wrk->next->prev = wrk->prev;
        wrk->prev->next = wrk->next;

        free (wrk);
    }
