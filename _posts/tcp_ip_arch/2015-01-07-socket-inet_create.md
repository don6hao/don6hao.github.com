---
layout: post
title: "TCP/IP-->Socket-->创建PF_INET协议族的套接字"
description: "inet_create"
category: TCP_IP.Architecture.Design.and.Implementation.in.Linux
tags: []
---

创建PF_INET协议族的套接字
---

用户层调用socket(family,type,protocol)函数（[socket函数执行流程](http://don6hao.github.io/blog/2014/12/31/socket-layer.html))，
若family等于PF_INET协议族的话，内核最终会调用inet_create(若type等于SOCK_STREAM,则inet_create->tcp_v4_init_sock)函数来创建套接字。

<p><img src="./../../../../../../pic/Figure_3.4.png" alt="Figure_3.2"
width="300" height="200" /> </p>

inetsw_array
---
inetsw_array包含支持PF_INET协议族的各种IP协议（TCP，UDP，RAW）的所有信息，在inet_create函数中将使用struct sock和struct socket来存储这些信息已方便当前套接字使用。

比如inetsw_array[0]的值

    {
            .type =       SOCK_STREAM,
            .protocol =   IPPROTO_TCP,
            .prot =       &tcp_prot,
            .ops =        &inet_stream_ops,
            .capability = -1,
            .no_check =   0,
            .flags =      INET_PROTOSW_PERMANENT |
              INET_PROTOSW_ICSK,
    },

用户层调用socket相关系统函数后，内核层首先会调用struct
socket->ops中的函数，然后在调用strcut sock->port中的函数。见下图
<p><img src="./../../../../../../pic/Figure_3.3.png" alt="Figure_3.3" width="800" height="400" /> </p>

inet_create函数
---
假设用户的命令是socket(PF_INET, SOCK_STREAM, 0/*IPPROTO_IP = 0 */),则inet_create的参数sock->type等于SOCK_STREAM, protocl等于0.
socket->type 等于 SOCK_STREAM， protocol等于0

    static int inet_create(struct socket *sock, int protocol)
    {
        struct sock *sk;
        struct list_head *p;
        struct inet_protosw *answer;
        struct inet_sock *inet;
        struct proto *answer_prot;
        unsigned char answer_flags;
        char answer_no_check;
        int try_loading_module = 0;
        int err;

        /*
         * SS_UNCONNECTED 处于未连接状态
         */
        sock->state = SS_UNCONNECTED;

        /* Look for the requested type/protocol pair. */
        answer = NULL;
    lookup_protocol:
        err = -ESOCKTNOSUPPORT;
        rcu_read_lock();
        list_for_each_rcu(p, &inetsw[sock->type]) {
            answer = list_entry(p, struct inet_protosw, list);

sock->type等于SOCK_STREAM, answer指向inetsw_array[0]

            /* Check the non-wild match. */
            if (protocol == answer->protocol) {
                if (protocol != IPPROTO_IP)
                    break;
            } else {

protocol等于IPPROTO_IP不等于IPPROTO_TCP(answer->protocl),把answer->protocol(IPPROTO_TCP)复制给protocol

                /* Check for the two wild cases. */
                if (IPPROTO_IP == protocol) {
                    protocol = answer->protocol;
                    break;
                }
                if (IPPROTO_IP == answer->protocol)
                    break;
            }
            err = -EPROTONOSUPPORT;
            answer = NULL;
        }

把answer(指向inetsw_array[socket->type])的信息复制给sock(struct socket)

        /*answer->prot = tcp_prot*/
        sock->ops = answer->ops;

        /* answer->ops  = &inet_stream_ops */
        answer_prot = answer->prot;

        answer_no_check = answer->no_check;
        answer_flags = answer->flags;
        rcu_read_unlock();

        BUG_TRAP(answer_prot->slab != NULL);

        err = -ENOBUFS;

分配一个sk(struct sock),sk->sk_prot = sk->sk_prot_creator=answer_prot(假设指向tcp_prot);

        sk = sk_alloc(PF_INET, GFP_KERNEL, answer_prot, 1);
        if (sk == NULL)
            goto out;

        err = 0;
        /* 计算校验和 */
        sk->sk_no_check = answer_no_check;
        if (INET_PROTOSW_REUSE & answer_flags)
            sk->sk_reuse = 1;
        
        /*
         *static inline struct inet_sock *inet_sk(const struct sock *sk)
         *{
         *   return (struct inet_sock *)sk;
         *}
         */
        inet = inet_sk(sk);
        /* INET_PROTOSW_ICSK:an inet_connection_sock */
        inet->is_icsk = INET_PROTOSW_ICSK & answer_flags;

        if (SOCK_RAW == sock->type) {
            inet->num = protocol;
            if (IPPROTO_RAW == protocol)
                inet->hdrincl = 1;
        }

        /* 混杂模式 */
        if (ipv4_config.no_pmtu_disc)
            inet->pmtudisc = IP_PMTUDISC_DONT;
        else
            inet->pmtudisc = IP_PMTUDISC_WANT;

        inet->id = 0;

sock_init_data函数初始化sk(struct sock)与IP协议相关联的部分，若sock不为空则进行sock->sk=sk操作

        sock_init_data(sock, sk);

        /* called for cleanup operations on the socket when it is destroyed. */
        sk->sk_destruct	   = inet_sock_destruct;
        sk->sk_family	   = PF_INET;
        sk->sk_protocol	   = protocol;
        sk->sk_backlog_rcv = sk->sk_prot->backlog_rcv;

        inet->uc_ttl	= -1;
        inet->mc_loop	= 1;
        inet->mc_ttl	= 1;
        inet->mc_index	= 0;
        inet->mc_list	= NULL;

        sk_refcnt_debug_inc(sk);

        if (inet->num) {
            /* It assumes that any protocol which allows
             * the user to assign a number at socket
             * creation time automatically
             * shares.
             */
            inet->sport = htons(inet->num);
            /* Add to protocol hash chains. */
            sk->sk_prot->hash(sk);
        }

从上面得知sk->sk_prot指向answer_prot.假设answer_prot指向tcp_prot，sk->sk_prot->init就会调用tcp_v4_init_sock(struct sock *sk)。tcp_v4_init_sock也是初始化一些变量和函数指针（比如send/receive
buffer的大小，定时器等）。

        if (sk->sk_prot->init) {
            err = sk->sk_prot->init(sk);
            if (err)
                sk_common_release(sk);
        }
    out:
        return err;
    out_rcu_unlock:
        rcu_read_unlock();
        goto out;
    }

inet_create函数的核心就是sk(struct sock)初始化，
它包含PF_INET协议族的相关函数操作集，指定协议套接字(TCP，UDP，RAW）的相关函数操作集，协议相关的数据结构的初始化等。
套接字初始化完毕后，就可以调用函数(bind, listen, accept等）来处理网络来的数据。
