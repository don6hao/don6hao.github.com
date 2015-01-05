---
layout: post
title: "Nginx-信号初始化"
description: "signal nginx"
category: Nginx
tags: []
---

信号初始化
---
>nginx启动的时候会调用ngx_init_signals函数遍历signals中的信号并进行信号处理函数注册

>ngx_signal_t结构体

    typedef struct {
        int     signo;
        char   *signame;
        char   *name;
        void  (*handler)(int signo);
    } ngx_signal_t;

>signals主要包含nginx需要处理的信号量和对应的信号处理函数

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

> ngx_init_signals 信号初始化函数

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

