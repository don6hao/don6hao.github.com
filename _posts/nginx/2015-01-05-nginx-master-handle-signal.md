---
layout: post
title: "Nginx-master进程的信号处理"
description: "master process signal nginx"
category: Nginx
tags: []
---

>master进程主要负责监控worker子进程务、重启服务、平滑升级、更换日志文件、配置文件实时生效等功能，不需要处理网络事件，不负责业务的执行。

master进程全貌图
---
![parent_handle_signal](./../../../../../../pic/parent_handle_signal.png) 

master进程中信号的定义:
---

|     信号     |        全局变量    |      含义          |
| -------------| ------------------ | ------------------ |
| QUIT         | ngx_quit           | 优雅地关闭整个服务 |
| TERM 或 INT  | ngx_terminate      | 强制关闭整个服务   |
| USR1         | ngx_reopen         | 重新打开服务中的所有文件 |
| WINCH        | ngx_noaccept       | 所有子进程不在accept连接，实际相当于对所有子进程发送QUIT信号 |
| USR2         | ngx_change_binary  | 平滑升级到新版本 |
| HUP          | ngx_reconfigure    | 重新读取配置文件 |
| CHLD         | ngx_reap           | 子进程意外结束，需要监控子进程 |




master进程如何处理信号
---
>处理部分主要在ngx_master_process_cycle函数的for循环中

    void ngx_master_process_cycle(ngx_cycle_t *cycle)
    {
        /*......*/
        sigemptyset(&set);
        /*......*/
        ngx_new_binary = 0;
        delay = 0;
        sigio = 0;
        live = 1;
        /* ...... */
    }


>**for循环开始部分主要处理：**

>1.设置定时器,定时发送SIGALRM信号

>2.接收SIGALRM信号后ngx_sigalrm值等于1，delay的值翻倍，延长定时器触发时间

>3.delay还有一个作用就是当接收到SIGINT信号后，delay用来判断等待子进程退出的时间是否超时




    for ( ;; ) {
        if (delay) {
            if (ngx_sigalrm) {
                sigio = 0;
                delay *= 2;
                ngx_sigalrm = 0;
            }

            /*
             *  struct itimerval {
             *      struct timeval it_interval;
             *      struct timeval it_value;
             *  };
             *  struct timeval {
             *      long tv_sec;
             *      long tv_usec;
             *  };
             *  it_interval指定间隔时间，it_value指定初始定时时间。
             *  如果只指定it_value，就是实现一次定时；
             *  如果同时指定 it_interval，则超时后，系统会重新初始化it_value为it_interval，实现重复定时；
             *  两者都清零，则会清除定时器。
             *
             */
            itv.it_interval.tv_sec = 0;
            itv.it_interval.tv_usec = 0;
            itv.it_value.tv_sec = delay / 1000;
            itv.it_value.tv_usec = (delay % 1000 ) * 1000;

            /* 设置定时器
             * ITIMER_REAL: 以系统真实的时间来计算，它送出SIGALRM信号。
             * ITIMER_VIRTUAL: -以该进程在用户态下花费的时间来计算，它送出SIGVTALRM信号。
             * ITIMER_PROF: 以该进程在用户态下和内核态下所费的时间来计算，它送出SIGPROF信号。
             * setitimer()调用成功返回0，否则返回-1。
             * */
            if (setitimer(ITIMER_REAL, &itv, NULL) == -1) {
                ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                              "setitimer() failed");
            }
        }


>**sigsuspend()函数调用**
    
>该函数调用使得master进程的大部分时间都处于挂起状态，直到master进程收到信号（SIGALRM或其它信号)为止。

    /*
     * 延时等待定时器
     * sigsuspend函数接受一个信号集指针，将信号屏蔽字设置为信号集中的值，
     * 在进程接受到一个信号之前，进程会挂起，当捕捉一个信号，
     * 首先执行信号处理程序，然后从sigsuspend返回，
     * 最后将信号屏蔽字恢复为调用sigsuspend之前的值。
     * 由于前面调用sigemptyset(&set);信号集位空，
     * sigsuspend(&set)不会阻塞任何信号，一直等到有信号发生才走下去
     *
     * */
    sigsuspend(&set);

    ngx_time_update();

>**接受GIGCHLD信号**

>有子进程意外结束，需要监控所有子进程

    /* 若ngx_reap为1，说明有子进程已退出 */
    if (ngx_reap) {
        ngx_reap = 0;
        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, "reap children");
        /*
         * 这个里面处理退出的子进程(有的worker异常退出，这时我们就需要重启这个worker )，
         * 如果所有子进程都退出则会返回0.
         */
        live = ngx_reap_children(cycle);
    }

>如果没有存活的子进程，并且收到了ngx_terminate或者ngx_quit信号，则master退出

    /* 当live标志位为0（表示所有子进程已经退出）、
     * ngx_terminate标志位为1或者ngx_quit标志位为1表示要退出master进程 
     */
    if (!live && (ngx_terminate || ngx_quit)) {
        ngx_master_process_exit(cycle);
    }

>接受到SIGINT信号,若超时强制关闭worker子进程

    /* 收到sigint 信号 */
    if (ngx_terminate) {
        if (delay == 0) {
            /* 设置延时 */
            delay = 50;
        }

        if (sigio) {
            sigio--;
            continue;
        }

        sigio = ccf->worker_processes + 2 /* cache processes */;

        if (delay > 1000) {
            /* 若超时，强制kill worker */
            ngx_signal_worker_processes(cycle, SIGKILL);
        } else {
            /* 负责发送sigint给worker，让它退出*/
            ngx_signal_worker_processes(cycle,
                                   ngx_signal_value(NGX_TERMINATE_SIGNAL));
        }

        continue;
    }

>**接收到SIGQUIT信号**
    
>关闭整个服务（子进程和套接字）

    /* 收到quit信号 */
    if (ngx_quit) {
        /* 发送给worker进程quit信号 */
        ngx_signal_worker_processes(cycle,
                                    ngx_signal_value(NGX_SHUTDOWN_SIGNAL));

        ls = cycle->listening.elts;
        for (n = 0; n < cycle->listening.nelts; n++) {
            if (ngx_close_socket(ls[n].fd) == -1) {
                ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_socket_errno,
                              ngx_close_socket_n " %V failed",
                              &ls[n].addr_text);
            }
        }
        cycle->listening.nelts = 0;

        continue;
    }

>**当收到SIGHUP信号:**
    
>重新读取配置文件

    /*
     * 当 nginx 接收到 HUP 信号，它会尝试先解析配置文件（如果指定配置文件，就使用指定的，否则使用默认的），
     * 成功的话，就应用新的配置文件（例如：重新打开日志文件或监听的套接字）。
     * 之后，nginx 运行新的工作进程并从容关闭旧的工作进程。
     * 通知工作进程关闭监听套接字但是继续为当前连接的客户提供服务。
     * 所有客户端的服务完成后，旧的工作进程被关闭。
     * 如果新的配置文件应用失败，nginx 将继续使用旧的配置进行工作。
     *
     */
    if (ngx_reconfigure) {
        ngx_reconfigure = 0;

        /*
         * 判断是否热代码替换后的新的代码还在运行中(也就是还没退出当前的master)。
         * 如果还在运行中，则不需要重新初始化config。
         */
        if (ngx_new_binary) {
            ngx_start_worker_processes(cycle, ccf->worker_processes,
                                       NGX_PROCESS_RESPAWN);
            ngx_start_cache_manager_processes(cycle, 0);
            ngx_noaccepting = 0;

            continue;
        }

        ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reconfiguring");

        /* 会尝试先解析配置文件（如果指定配置文件，就使用指定的，否则使用默认的）
         * 成功的话，就应用新的配置文件（例如：重新打开日志文件或监听的套接字）。
         * */
        cycle = ngx_init_cycle(cycle);
        if (cycle == NULL) {
            cycle = (ngx_cycle_t *) ngx_cycle;
            continue;
        }

        /* 使用新的配置文件，并重新启动新的worker */
        ngx_cycle = cycle;
        ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx,
                                               ngx_core_module);
        ngx_start_worker_processes(cycle, ccf->worker_processes,
                                   NGX_PROCESS_JUST_RESPAWN);
        ngx_start_cache_manager_processes(cycle, 1);

        /* allow new processes to start */
        ngx_msleep(100);

        live = 1;
        /* nginx 运行新的工作进程并从容关闭旧的工作进程 */
        ngx_cycle = cycle;
        ngx_signal_worker_processes(cycle,
                                    ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
    }

>重启worker子进程,ngx_restart标志位与信号无关

>ngx_restart标志位在ngx_noaccepting（表示正在停止接受新的连接）为1的时候被设置为1

    /*
     * 代码里面是当热代码替换后，如果ngx_noacceptig被设置了，
     * 则设置这个标志位(难道意思是热代码替换前要先停止当前的accept连接？)
     *
     */
    if (ngx_restart) {
        ngx_restart = 0;
        ngx_start_worker_processes(cycle, ccf->worker_processes,
                                   NGX_PROCESS_RESPAWN);
        ngx_start_cache_manager_processes(cycle, 0);
        live = 1;
    }

>**收到USR1信号:**
    
>重新打开服务中的所有文件

    /* 重新打开log */
    if (ngx_reopen) {
        ngx_reopen = 0;
        ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reopening logs");
        ngx_reopen_files(cycle, ccf->user);
        ngx_signal_worker_processes(cycle,
                                    ngx_signal_value(NGX_REOPEN_SIGNAL));
    }

>**收到USR2信号:**
    
>平滑升级到新版本

    /* 热代码替换
     * 在不中断服务的情况下 - 新的请求也不会丢失，
     * 使用新的 nginx 可执行程序替换旧的（当升级新版本或添加/删除服务器模块时）。
     * 两个 nginx 实例会同时运行，一起处理输入的请求。
     * 要逐步停止旧的实例，你必须发送 WINCH 信号给旧的主进程
     *
     * */
    if (ngx_change_binary) {
        ngx_change_binary = 0;
        ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "changing binary");
        /* 进行热代码替换，这里是调用execve来执行新的代码 */
        ngx_new_binary = ngx_exec_new_binary(cycle, ngx_argv);
    }

>**收到WINCH信号:**
    
>所有子进程不再accept,关闭worker子进程
    
    /* 让worker进程停止接受accept连接，并让worker进程从容关闭 */
    if (ngx_noaccept) {
        ngx_noaccept = 0;
        ngx_noaccepting = 1;
        ngx_signal_worker_processes(cycle,
                                    ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
    }

