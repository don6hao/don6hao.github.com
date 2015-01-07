---
layout: post
title: "Nginx--信号"
description: "signal nginx"
category: Nginx
tags: []
---

信号的初始化
---
>nginx启动的时候会调用ngx_init_signals函数遍历signals中的信号并进行信号处理函数注册

signals结构体
---

>ngx_signal_t结构体

    typedef struct {
        int     signo;
        char   *signame;
        char   *name;
        void  (*handler)(int signo);
    } ngx_signal_t;

signals初始化
---

>主要包含nginx需要处理的信号量和对应的信号处理函数（ngx_signal_handler或者SIG_IGN)

    ngx_signal_t  signals[] = {
        { ngx_signal_value(NGX_RECONFIGURE_SIGNAL),
          "SIG" ngx_value(NGX_RECONFIGURE_SIGNAL),
          "reload",
          ngx_signal_handler },

        { ngx_signal_value(NGX_REOPEN_SIGNAL),
          "SIG" ngx_value(NGX_REOPEN_SIGNAL),
          "reopen",
          ngx_signal_handler },

        { ngx_signal_value(NGX_NOACCEPT_SIGNAL),
          "SIG" ngx_value(NGX_NOACCEPT_SIGNAL),
          "",
          ngx_signal_handler },

        { ngx_signal_value(NGX_TERMINATE_SIGNAL),
          "SIG" ngx_value(NGX_TERMINATE_SIGNAL),
          "stop",
          ngx_signal_handler },

        { ngx_signal_value(NGX_SHUTDOWN_SIGNAL),
          "SIG" ngx_value(NGX_SHUTDOWN_SIGNAL),
          "quit",
          ngx_signal_handler },

        { ngx_signal_value(NGX_CHANGEBIN_SIGNAL),
          "SIG" ngx_value(NGX_CHANGEBIN_SIGNAL),
          "",
          ngx_signal_handler },

        { SIGALRM, "SIGALRM", "", ngx_signal_handler },

        { SIGINT, "SIGINT", "", ngx_signal_handler },

        { SIGIO, "SIGIO", "", ngx_signal_handler },

        { SIGCHLD, "SIGCHLD", "", ngx_signal_handler },

        { SIGSYS, "SIGSYS, SIG_IGN", "", SIG_IGN },

        { SIGPIPE, "SIGPIPE, SIG_IGN", "", SIG_IGN },

        { 0, NULL, "", NULL }
    };

ngx_init_signals
---

> 信号初始化，遍历signals,注册信号处理函数

    ngx_int_t
    ngx_init_signals(ngx_log_t *log)
    {
        ngx_signal_t      *sig;
        struct sigaction   sa;

        /* 遍历signals,注册信号处理函数 */
        for (sig = signals; sig->signo != 0; sig++) {
            ngx_memzero(&sa, sizeof(struct sigaction));
            sa.sa_handler = sig->handler;
            sigemptyset(&sa.sa_mask);
            if (sigaction(sig->signo, &sa, NULL) == -1) {
                /* ...... */
            }
        }

        return NGX_OK;
    }


ngx_signal_handler
---

>信号处理函数,主要分为两个部分：

>1.MASTER/SINGLE进程的信号处理，接收到某信号后设置对应的全局标志位

>比如收到QUIT信号后，设置ngx_terminate的值为1

>2.WORKER/HELPER进程的信号处理，接收到某信号后设置对应的全局标志位

    void
    ngx_signal_handler(int signo)
    {
        char            *action;
        ngx_int_t        ignore;
        ngx_err_t        err;
        ngx_signal_t    *sig;

        ignore = 0;

        err = ngx_errno;

        for (sig = signals; sig->signo != 0; sig++) {
            if (sig->signo == signo) {
                break;
            }
        }

        ngx_time_sigsafe_update();

        action = "";

        switch (ngx_process) {

>**master进程的信号处理**

        case NGX_PROCESS_MASTER:
        case NGX_PROCESS_SINGLE:
            switch (signo) {

            case ngx_signal_value(NGX_SHUTDOWN_SIGNAL):
                /* 如果接受到quit信号，则准备退出进程。*/
                ngx_quit = 1;
                action = ", shutting down";
                break;

            case ngx_signal_value(NGX_TERMINATE_SIGNAL):
            case SIGINT:
                /* 如果接受到quit信号，则准备kill worker进程。*/
                ngx_terminate = 1;
                action = ", exiting";
                break;
            /* winch信号，停止接受accept */
            case ngx_signal_value(NGX_NOACCEPT_SIGNAL):
                if (ngx_daemonized) {
                    ngx_noaccept = 1;
                    action = ", stop accepting connections";
                }
                break;
            /* sighup信号用来reconfig */
            case ngx_signal_value(NGX_RECONFIGURE_SIGNAL):
                ngx_reconfigure = 1;
                action = ", reconfiguring";
                break;
            /* 用户信号 ，重新打开log */
            case ngx_signal_value(NGX_REOPEN_SIGNAL):
                ngx_reopen = 1;
                action = ", reopening logs";
                break;
            /* 热代码替换 */
            case ngx_signal_value(NGX_CHANGEBIN_SIGNAL):
                if (getppid() > 1 || ngx_new_binary > 0) {

                    /*
                     * Ignore the signal in the new binary if its parent is
                     * not the init process, i.e. the old binary's process
                     * is still running.  Or ignore the signal in the old binary's
                     * process if the new binary's process is already running.
                     * 若进程的父进程（old 代码进程是父进程）不是init进程（getppid==1)，忽略此信号。
                     * 新的代码进程已在运行中（ngx_new_binary > 0)，old代码进程忽略信号，
                     */

                    action = ", ignoring";
                    ignore = 1;
                    break;
                }
                /* 正常情况下，需要热代码替换。设置标志位 */
                ngx_change_binary = 1;
                action = ", changing binary";
                break;

            case SIGALRM:
                ngx_sigalrm = 1;
                break;

            case SIGIO:
                ngx_sigio = 1;
                break;
            /* 子进程退出 */
            case SIGCHLD:
                ngx_reap = 1;
                break;
            }

            break;

>**worker进程的信号处理**

        /* worker的信号处理 */
        case NGX_PROCESS_WORKER:
        case NGX_PROCESS_HELPER:
            switch (signo) {

            case ngx_signal_value(NGX_NOACCEPT_SIGNAL):
                if (!ngx_daemonized) {
                    break;
                }
                ngx_debug_quit = 1;
            case ngx_signal_value(NGX_SHUTDOWN_SIGNAL):
                ngx_quit = 1;
                action = ", shutting down";
                break;

            case ngx_signal_value(NGX_TERMINATE_SIGNAL):
            case SIGINT:
                ngx_terminate = 1;
                action = ", exiting";
                break;

            case ngx_signal_value(NGX_REOPEN_SIGNAL):
                ngx_reopen = 1;
                action = ", reopening logs";
                break;

            case ngx_signal_value(NGX_RECONFIGURE_SIGNAL):
            case ngx_signal_value(NGX_CHANGEBIN_SIGNAL):
            case SIGIO:
                action = ", ignoring";
                break;
            }

            break;
        }

        ngx_log_error(NGX_LOG_NOTICE, ngx_cycle->log, 0,
                      "signal %d (%s) received%s", signo, sig->signame, action);

        if (ignore) {
            ngx_log_error(NGX_LOG_CRIT, ngx_cycle->log, 0,
                          "the changing binary signal is ignored: "
                          "you should shutdown or terminate "
                          "before either old or new binary's process");
        }

        /*
         * 最终如果信号是sigchld，我们收割僵尸进程(用waitpid)
         */
        if (signo == SIGCHLD) {
            ngx_process_get_status();
        }

        ngx_set_errno(err);
    }

