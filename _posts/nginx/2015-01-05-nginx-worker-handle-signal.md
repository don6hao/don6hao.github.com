---
layout: post
title: "Nginx--worker进程的信号处理"
description: "worker process signal nginx"
category: Nginx
tags: []
---


信号处理
---
>worker信号主要是[ngx_signal_handler函数](http://don6hao.github.io/nginx/2015/01/04/nginx-init-signal/)来处理，它接收到某信号后会设置对应的全局标志位。

>worker进程会调用ngx_worker_process_cycle函数循环检测标志位并进行对应操作。

ngx_worker_process_cycle函数
---

>ngx_worker_process_cycle函数主要关注4个全局标志位：

>
|        标志位      |      含义          |
| ------------------ | ------------------ |
| ngx_quit           | 优雅地关闭整个服务 |
| ngx_terminate      | 强制关闭整个服务   |
| ngx_reopen         | 重新打开服务中的所有文件 |
| ngx_exiting        | 退出进程标志位 |

    static void
    ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
    {
        ngx_int_t worker = (intptr_t) data;

        ngx_uint_t         i;
        ngx_connection_t  *c;

        ngx_process = NGX_PROCESS_WORKER;

        ngx_worker_process_init(cycle, worker);

        ngx_setproctitle("worker process");

        for ( ;; ) {

>**ngx_exiting 标志位**
    
>对进程的连接进行清理并进程退出

            if (ngx_exiting) {

                c = cycle->connections;

                for (i = 0; i < cycle->connection_n; i++) {

                    /* THREAD: lock */

                    if (c[i].fd != -1 && c[i].idle) {
                        c[i].close = 1;
                        c[i].read->handler(c[i].read);
                    }
                }

                if (ngx_event_timer_rbtree.root == ngx_event_timer_rbtree.sentinel)
                {
                    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");

                    ngx_worker_process_exit(cycle);
                }
            }

            ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, "worker cycle");

            ngx_process_events_and_timers(cycle);

> **SIGINT信号**

> 检测到SIGINT信号，调用ngx_worker_process进程退出

            if (ngx_terminate) {
                ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");

                ngx_worker_process_exit(cycle);
            }

> **SIGQUIT信号**

> 检测到SIGQUIT信号, 管理套接字设置ngx_exiting标志位

            if (ngx_quit) {
                ngx_quit = 0;
                ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0,
                              "gracefully shutting down");
                ngx_setproctitle("worker process is shutting down");

                if (!ngx_exiting) {
                    ngx_close_listening_sockets(cycle);
                    ngx_exiting = 1;
                }
            }

> **SIGINT信号**

> 检测到SIGINT信号，重新打开文件

            if (ngx_reopen) {
                ngx_reopen = 0;
                ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reopening logs");
                ngx_reopen_files(cycle, -1);
            }
        }
    }

