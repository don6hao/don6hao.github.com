---
layout: post
title: "TCP/IP-->Socket-->inetsw和inet_protosw"
description: "inetsw struct inet_protosw"
category: TCP_IP.Architecture.Design.and.Implementation.in.Linux
tags: []
---

**inetsw结构体**
---
>inetsw是指向链表的数组，每个数组成员都指向的链表代表不同的套接字类型,比如TCP，UDP或RAW等。

    {% highlight ruby %}
    enum sock_type {
        SOCK_DGRAM	= 1,
        SOCK_STREAM	= 2,
        SOCK_RAW	= 3,
        SOCK_RDM	= 4,
        SOCK_SEQPACKET	= 5,
        SOCK_DCCP	= 6,
        SOCK_PACKET	= 10,
    };

    #define SOCK_MAX (SOCK_PACKET + 1)

    static struct list_head inetsw[SOCK_MAX];
    {% endhighlight %}

>inetsw在inet_init函数中初始化

	/* Register the socket-side information for inet_create. */
	for (r = &inetsw[0]; r < &inetsw[SOCK_MAX]; ++r)
		INIT_LIST_HEAD(r);

**inetsw_array结构体**
---
        
>包含PF_INET协议族的所有套接字类型的信息。INET套接字类型：SOCK_STREAM, SOCK_DGRAM, SOCK_RAW

    static struct inet_protosw inetsw_array[] =
    {
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

            {
                    .type =       SOCK_DGRAM,
                    .protocol =   IPPROTO_UDP,
                    .prot =       &udp_prot,
                    .ops =        &inet_dgram_ops,
                    .capability = -1,
                    .no_check =   UDP_CSUM_DEFAULT,
                    .flags =      INET_PROTOSW_PERMANENT,
           },
            

           {
                   .type =       SOCK_RAW,
                   .protocol =   IPPROTO_IP,	/* wild card */
                   .prot =       &raw_prot,
                   .ops =        &inet_sockraw_ops,
                   .capability = CAP_NET_RAW,
                   .no_check =   UDP_CSUM_DEFAULT,
                   .flags =      INET_PROTOSW_REUSE,
           }
    };

>在inet_init函数中调用inet_register_protosw函数把inetsw_array的成员插入到inetsw对应的链表中

	for (q = inetsw_array; q < &inetsw_array[INETSW_ARRAY_LEN]; ++q)
		inet_register_protosw(q);

>inetsw_array的每个成员根据.type插入到inetsw[type]的链表中

    #define INETSW_ARRAY_LEN (sizeof(inetsw_array) / sizeof(struct inet_protosw))

    void inet_register_protosw(struct inet_protosw *p)
    {
        struct list_head *lh;
        struct inet_protosw *answer;
        int protocol = p->protocol;
        struct list_head *last_perm;

        spin_lock_bh(&inetsw_lock);

        if (p->type >= SOCK_MAX)
            goto out_illegal;

        /* If we are trying to override a permanent protocol, bail. */
        answer = NULL;
        last_perm = &inetsw[p->type];
        list_for_each(lh, &inetsw[p->type]) {

            /* 通过list_entry计算出struct inet_protosw的地址
             * inet_protosw的成员list地址减去list在inet_protosw中的偏移量
             * 得到inet_protosw的地址
             */
            answer = list_entry(lh, struct inet_protosw, list);

            /* Check only the non-wild match. */
            /* 检查是否有已经注册过的socket type
             * 若flags中的INET_PROTOSW_PERMANENT(表示不能被移除的协议)位等于1
             * 且protocol相等，不能覆盖已有的socket类型.
             * 直接break跳出循环，answer不等于NULL，跳转到out_permanent
             */
            if (INET_PROTOSW_PERMANENT & answer->flags) {
                if (protocol == answer->protocol)
                    break;
                last_perm = lh;
            }

            answer = NULL;
        }
        if (answer)
            goto out_permanent;

        /* Add the new entry after the last permanent entry if any, so that
         * the new entry does not override a permanent entry when matched with
         * a wild-card protocol. But it is allowed to override any existing
         * non-permanent entry.  This means that when we remove this entry, the 
         * system automatically returns to the old behavior.
         * 挂链操作
         */
        list_add_rcu(&p->list, last_perm);
    out:
        spin_unlock_bh(&inetsw_lock);

        synchronize_net();

        return;

    out_permanent:
        printk(KERN_ERR "Attempt to override permanent protocol %d.\n",
               protocol);
        goto out;

    out_illegal:
        printk(KERN_ERR
               "Ignoring attempt to register invalid socket type %d.\n",
               p->type);
        goto out;
    }

