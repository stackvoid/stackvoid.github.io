---
layout: post
title: bluedroid源码分析之ACL包发送和接收（四）
categories: [bluedroid]
tags: [ACL]
---

本文分析 L2CAP 底层的数据包发送函数。

![ACL02](/album/l2c_link_check_send_pkts2.png)

**l2c_link_check_send_pkts (tL2C_LCB *p_lcb, tL2C_CCB *p_ccb, BT_HDR *p_buf)** 函数的主要作用是：

1. 如果 p_buf 不为 null，p_ccb 为null(在signaling状态中，信道是Fixed的) 说明这个 ACL 包是这类 L2CAP 的 command：command reject、configuration request、connection request、connection response、connection response neg、configuration response、configuration reject、disconnect response、echo request、echo response、info request、info response、disconnect request，或者说明这个 ACL 包是 S_frame(Supervisory Frame), 目前 Bluedroid 中只有这两种情况；然后把这个包放到 p_lcb->link_xmit_data_q 中，注意是当前 Link 上的 link_xmit_data_q，跟 CCB 中的 xmit_hold_q 队列是完全不同的两个队列。
2. 检查 l2cb.is_cong_cback_context 字段，如果为 TRUE，说明当前 Link 拥堵，当前发送过程结束，注意，这时候 ACL 包都已经进入队列了，都在等待发送(即在此调用本函数)。
3. 若 p_lcb 不为 null 或 p_lcb->link_xmit_quota 不为0：
    - 首先检查 p_lcb 的 partial_segment_being_sent(为TRUE，说明 Segment还没发完，底层来保证，所以当前函数返回)
    - p_lcb 的 link_state 不为 LST_CONNECTED，说明当前 Link 异常，当前函数返回
    - L2C_LINK_CHECK_POWER_MODE 默认是关闭的，关于电源管理的，暂时不分析
    - 步骤 1 分析知道，我们现在发送的是 当前 Link 上的 link_xmit_data_q 队列里的包，对于 BR/EDR来说，如果条件 l2cb.controller_xmit_window 不为 0（否则没有窗口发送数据包了）并且 p_lcb->sent_not_acked < p_lcb->link_xmit_quota(没回复的包一定要小于当前Link最大发送值)，不成立，那么在这里轮询，直到有了发送窗口并且 sent_not_acked < link_xmit_quota 才将数据包交给 l2c_link_send_to_lower 函数做进一步处理。 
    - 这时，Link 上的数据包已经发送了，若 single_write(断开前最后CCB中的数据处理标志)为false，我们要继续去看看当前 Link 上的每个 Channel 上的 Queue 中有没有数据包，一旦找到一个 p_buf 数据包，就发送(不会循环找，找到了就返回)

OK，上面流程 1 -> 3 是一个单独的流程, 处理一些 L2CAP command 或 Response 的发送情况。

下面偶在来分析另外一个 case，这**三个参数全部为非空**的情况，这种情况只在 L2CAP 发送 disconnect request 命令时出现(关闭当前 Link 要确保对应上层应用的 CCB 中的数据要发完)，主要作用是：

1. 在函数 l2cu_send_peer_disc_req 中我们看到 CCB 中的数据包直接从 CCB 的 xmit_hold_q 中出队列，设置 ACL 包头就直接交给 l2c_link_check_send_pkts(p_ccb->p_lcb, p_ccb, p_buf2)发送了。
2. 看到 l2c_link_check_send_pkts 的三个参数均不为 null，说明我们把当前的 CCB 中 xmit_hold_q 的数据包 当成 Link 上的 link_xmit_data_q 发送出去了(相当于优先发送了)，single_write 是一个标志位，代表当前数据包由 CCB -> LCB 代替发送。由于判断了 single_write 这个标志位，程序不会再去 CCB 中找数据包了。

还有一种 case 是**全部为空**的情况，这是触发了 RR 机制所致，正常情况还有发送窗口的情况下，会遍历每个 Link ，然后先发当前 Link上的 Queue里的数据发送出去，然后再发送 CCB 中 Queue 的数据。

OK，这个函数还剩最后一个 case，就是**正常情况**下，我们听音乐的数据流发送的情况，这是最常见的一种情况，调用形式为 l2c_link_check_send_pkts (p_lcb, NULL, NULL)。

1. 判断当前 Link 是否拥堵。
2. 首先看看 Link 上的 link_xmit_data_q 队列有没有数据，有的话发送
3. 如果 link_xmit_data_q 没有数据，在看看  Link 上的 CCB，通过 CCB 的链表，找到 CCB 上的队列，一直找，直到找到一个 数据包，发送之。
4. 注意：上层在发送数据包，仅仅是调用一下这个函数，至于 是不是这个数据包，那不一定；反正每次上层调用都会发送一个，上层发下来的包总会发出去的。

{% highlight c linenos %}
void l2c_link_check_send_pkts (tL2C_LCB *p_lcb, tL2C_CCB *p_ccb, BT_HDR *p_buf)
{
    int         xx;
    BOOLEAN     single_write = FALSE; //最后 Link Disc 用来把 CCB 中的数据包放到 Link 上的队列发，速度加快
    L2CAP_TRACE_DEBUG0("mike: in func-- l2c_link_check_send_pkts");
    /* Save the channel ID for faster counting */
    if (p_buf) //一般数据包都为空，只有发送 L2CAP 发送 command/response 或 发送 S-Frame 才用到
    {
        if (p_ccb != NULL) //这个 case 就是 当前 Link 即将断开的情况了
        {
            p_buf->event = p_ccb->local_cid;
            single_write = TRUE; //见上面注释
            L2CAP_TRACE_DEBUG0("mike: l2c_link_check_send_pkts-- p_buf p_ccb not null");
        }
        else
            p_buf->event = 0;

        p_buf->layer_specific = 0;
        L2CAP_TRACE_DEBUG0("mike: l2c_link_check_send_pkts-- p_buf->layer_specific=0");
        GKI_enqueue (&p_lcb->link_xmit_data_q, p_buf); //把这个数据包放到 当前 link 上的 link_xmit_data_q队列中

        if (p_lcb->link_xmit_quota == 0){
            l2cb.check_round_robin = TRUE; // 没有发送窗口了，需要 RR 看看有没有别的数据包可以发送的
            L2CAP_TRACE_DEBUG0("mike: l2c_link_check_send_pkts-- p_lcb->link_xmit_quota");
        }
    }

    /* If this is called from uncongested callback context break recursive calling.
    ** This LCB will be served when receiving number of completed packet event.
    */
    if (l2cb.is_cong_cback_context){//当前 Link 拥堵了，不发送数据包直接返回，几乎不会发生
        L2CAP_TRACE_DEBUG0("mike: l2c_link_check_send_pkts-- l2cd.is_cong_cback_context");
        return;
    }
    /* If we are in a scenario where there are not enough buffers for each link to
    ** have at least 1, then do a round-robin for all the LCBs
    */
    if ( (p_lcb == NULL) || (p_lcb->link_xmit_quota == 0) )
    {
        L2CAP_TRACE_DEBUG0("mike: l2c_link_check_send_pkts-- (p_lcb == NULL) ||(p_lcb->link_xmit_quota == 0)");
        if (p_lcb == NULL)
            p_lcb = l2cb.lcb_pool;
        else if (!single_write)
            p_lcb++;

        /* Loop through, starting at the next */
		//哎呀，没有足够发送窗口了，在所有的 Link 上做一次 RR
        for (xx = 0; xx < MAX_L2CAP_LINKS; xx++, p_lcb++)
        {
            L2CAP_TRACE_DEBUG1("mike: l2c_link_check_send_pkts--Loop through: xx = %d",xx);
            /* If controller window is full, nothing to do */
            if ( (l2cb.controller_xmit_window == 0
#if (BLE_INCLUDED == TRUE)
                  && !p_lcb->is_ble_link
#endif
                )
#if (BLE_INCLUDED == TRUE)
                || (p_lcb->is_ble_link && l2cb.controller_le_xmit_window == 0 )
#endif
              || (l2cb.round_robin_unacked >= l2cb.round_robin_quota) )
                break;

            /* Check for wraparound */
            if (p_lcb == &l2cb.lcb_pool[MAX_L2CAP_LINKS])
                p_lcb = &l2cb.lcb_pool[0];

            if ( (!p_lcb->in_use)
               || (p_lcb->partial_segment_being_sent)
               || (p_lcb->link_state != LST_CONNECTED)
               || (p_lcb->link_xmit_quota != 0)
               || (L2C_LINK_CHECK_POWER_MODE (p_lcb)) )
                continue;

            //首先从 当前 Link 上的 link_xmit_data_q 中取出数据包并发送  
            if ((p_buf = (BT_HDR *)GKI_dequeue (&p_lcb->link_xmit_data_q)) != NULL)
            {
                L2CAP_TRACE_DEBUG0("mike: l2c_link_check_send_pkts--if ((p_buf = (BT_HDR*)GKI_dequeue (&p_lcb->link_xmit_data_q)) != NULL)");
                l2c_link_send_to_lower (p_lcb, p_buf);
            }
            else if (single_write) //如果是 single_write 设置为 TRUE，说明数据包 已经在 link_xmit_data_q 发送了，没必要在执行下面的 code 了
            {
                /* If only doing one write, break out */
                L2CAP_TRACE_DEBUG0("mike: l2c_link_check_send_pkts--single write is true then break");
                break;
            }
            //Link 上的 Queue 中没有东西可以发送，查找 CCB 中的 Queue，直到找到一个为止。
            else if ((p_buf = l2cu_get_next_buffer_to_send (p_lcb)) != NULL)
            {
                L2CAP_TRACE_DEBUG0("mike: l2c_link_check_send_pkts--(p_buf=l2cu_get_next_buffer_to_send (p_lcb)) != NULL");
                l2c_link_send_to_lower (p_lcb, p_buf);
            }
        }

        /* If we finished without using up our quota, no need for a safety check */
#if (BLE_INCLUDED == TRUE)
        if ( ((l2cb.controller_xmit_window > 0 && !p_lcb->is_ble_link) ||
             (l2cb.controller_le_xmit_window > 0 && p_lcb->is_ble_link))
          && (l2cb.round_robin_unacked < l2cb.round_robin_quota) )
#else
        if ( (l2cb.controller_xmit_window > 0)
          && (l2cb.round_robin_unacked < l2cb.round_robin_quota) )

#endif
            l2cb.check_round_robin = FALSE;
    }
    else /* if this is not round-robin service */
    {
        /* If a partial segment is being sent, can't send anything else */
        if ( (p_lcb->partial_segment_being_sent)
          || (p_lcb->link_state != LST_CONNECTED)
          || (L2C_LINK_CHECK_POWER_MODE (p_lcb)) )
            return;

        /* See if we can send anything from the link queue */
#if (BLE_INCLUDED == TRUE)
        while ( ((l2cb.controller_xmit_window != 0 && !p_lcb->is_ble_link) ||
                 (l2cb.controller_le_xmit_window != 0 && p_lcb->is_ble_link))
             && (p_lcb->sent_not_acked < p_lcb->link_xmit_quota))
#else
        while ( (l2cb.controller_xmit_window != 0)
             && (p_lcb->sent_not_acked < p_lcb->link_xmit_quota))
#endif
        {
            if ((p_buf = (BT_HDR *)GKI_dequeue (&p_lcb->link_xmit_data_q)) == NULL)//发送Link上的数据包
                break;

            if (!l2c_link_send_to_lower (p_lcb, p_buf))
                break;
        }

        if (!single_write)//确保不是在 链路 disc 状态下
        {
            /* See if we can send anything for any channel */
#if (BLE_INCLUDED == TRUE)
            while ( ((l2cb.controller_xmit_window != 0 && !p_lcb->is_ble_link) ||
                    (l2cb.controller_le_xmit_window != 0 && p_lcb->is_ble_link))
                    && (p_lcb->sent_not_acked < p_lcb->link_xmit_quota))
#else
            while ((l2cb.controller_xmit_window != 0) && (p_lcb->sent_not_acked < p_lcb->link_xmit_quota))
#endif
            {
                if ((p_buf = l2cu_get_next_buffer_to_send (p_lcb)) == NULL)//找到一个数据包来发送
                    break;

                if (!l2c_link_send_to_lower (p_lcb, p_buf))
                    break;
            }
        }

        /* There is a special case where we have readjusted the link quotas and  */
        /* this link may have sent anything but some other link sent packets so  */
        /* so we may need a timer to kick off this link's transmissions.         */
        if ( (p_lcb->link_xmit_data_q.count) && (p_lcb->sent_not_acked < p_lcb->link_xmit_quota) )
            L2CAP_TRACE_DEBUG0("mike: l2c_link_check_send_pkts--a timer to kick off this link's transmissions");
            btu_start_timer (&p_lcb->timer_entry, BTU_TTYPE_L2CAP_LINK, L2CAP_LINK_FLOW_CONTROL_TOUT);
    }

}

{% endhighlight %}

最终 l2c_link_check_send_pkts 把数据包交给了 l2c_link_send_to_lower 来做处理，我们的音乐数据包最终也被从某个 CCB 中的队列出队列给了 l2c_link_send_to_lower。l2c_link_send_to_lower 主要做了这些事情：

1. 如果当前数据包 p_buf 的长度小于 ACL 包的最大值，sent_not_acked 加1，整个 L2CAP 的 controller_xmit_window 减1。然后通过 L2C_LINK_SEND_ACL_DATA 将此数据包发送出去。
2. 如果当前数据包 p_buf 的长度大于 ACL 包的最大值，先看看能分成几个分包(为了求的几个窗口能容下)，然后窗口值减掉这些分包个数，然后将整个数据包交给 L2C_LINK_SEND_ACL_DATA (大于ACL包长度)，具体分包发送由 H5(串口) 部分来负责。

{% highlight c linenos %}

/*******************************************************************************
**
** Function         l2c_link_send_to_lower
**
** Description      This function queues the buffer for HCI transmission
**
** Returns          TRUE for success, FALSE for fail
**
*******************************************************************************/
static BOOLEAN l2c_link_send_to_lower (tL2C_LCB *p_lcb, BT_HDR *p_buf)
{
    UINT16      num_segs;
    UINT16      xmit_window, acl_data_size;
    L2CAP_TRACE_DEBUG0("mike: l2c_link_send_to_lower");
#if (BLE_INCLUDED == TRUE)
    if ((!p_lcb->is_ble_link && (p_buf->len <= btu_cb.hcit_acl_pkt_size)) ||
        (p_lcb->is_ble_link && (p_buf->len <= btu_cb.hcit_ble_acl_pkt_size)))
#else
    if (p_buf->len <= btu_cb.hcit_acl_pkt_size) //一般都是走这条路径，p_buf一般不会超过 ACL 长度最大值
#endif
    {
        if (p_lcb->link_xmit_quota == 0){ // Link 上没有窗口了，controller_xmit_window 窗口还是有的，此时就是 round_roubin_unack了（因为上面的数据已经下来了，必须得发送）
            L2CAP_TRACE_DEBUG0("mike: l2c_link_send_to_lower--if (p_lcb->link_xmit_quota == 0)");
            l2cb.round_robin_unacked++;
            L2CAP_TRACE_DEBUG1("mike: l2c_link_send_to_lower--l2cb.round_robin_unacked=%d",l2cb.round_robin_unacked);
        }
        p_lcb->sent_not_acked++; //整个 Link 已经发送但是没有回复的数据包个数
        L2CAP_TRACE_DEBUG1("mike:l2c_link_send_to_lower--p_lcb->sent_not_acked:",p_lcb->sent_not_acked);
        p_buf->layer_specific = 0;

#if (BLE_INCLUDED == TRUE)
        if (p_lcb->is_ble_link)
        {
            l2cb.controller_le_xmit_window--;
            L2C_LINK_SEND_BLE_ACL_DATA (p_buf);
        }
        else
#endif
        {
            l2cb.controller_xmit_window--; //当前 controller 发送窗口减1
            L2CAP_TRACE_DEBUG1("mike:l2c_link_send_to_lower--,l2cb.controller_xmit_window=%d",l2cb.controller_xmit_window);
            L2CAP_TRACE_DEBUG0("mike: l2c_link_send_to_lower--L2C_LINK_SEND_ACL_DATA");
            L2C_LINK_SEND_ACL_DATA (p_buf); //发送当前这个数据包
        }
    }
    else
    {
#if BLE_INCLUDED == TRUE
        if (p_lcb->is_ble_link)
        {
            acl_data_size = btu_cb.hcit_ble_acl_data_size;
            xmit_window = l2cb.controller_le_xmit_window;

        }
        else
#endif
        {
            acl_data_size = btu_cb.hcit_acl_data_size;//ACL 包额度最大值
            xmit_window = l2cb.controller_xmit_window; //controller目前为止的可用窗口
        }
        num_segs = (p_buf->len - HCI_DATA_PREAMBLE_SIZE + acl_data_size - 1) / acl_data_size;
        L2CAP_TRACE_DEBUG3("mike: l2c_link_send_to_lower-- num_segs:%d, acl_data_size:%d,xmit_window=%d", num_segs,acl_data_size, xmit_window);

        /* If doing round-robin, then only 1 segment each time */
        if (p_lcb->link_xmit_quota == 0)
        {
            num_segs = 1;
            p_lcb->partial_segment_being_sent = TRUE;
        }
        else
        {
            /* Multi-segment packet. Make sure it can fit */
            if (num_segs > xmit_window)
            {
                num_segs = xmit_window;//分包个数比 controller 窗口的个数还多，只能发 controller 个数的包
                p_lcb->partial_segment_being_sent = TRUE; //标志位，还有分包，需要继续发送，Btu_task 中有个 Event 就是处理分包的
            }

            if (num_segs > (p_lcb->link_xmit_quota - p_lcb->sent_not_acked))
            {
                num_segs = (p_lcb->link_xmit_quota - p_lcb->sent_not_acked);
                p_lcb->partial_segment_being_sent = TRUE;
            }
        }

        p_buf->layer_specific        = num_segs;
#if BLE_INCLUDED == TRUE
        if (p_lcb->is_ble_link)
        {
            l2cb.controller_le_xmit_window -= num_segs;

        }
        else
#endif
        l2cb.controller_xmit_window -= num_segs;//分包占用的窗口数

        if (p_lcb->link_xmit_quota == 0)
            l2cb.round_robin_unacked += num_segs;

        p_lcb->sent_not_acked += num_segs;
#if BLE_INCLUDED == TRUE
        if (p_lcb->is_ble_link)
        {
            L2C_LINK_SEND_BLE_ACL_DATA(p_buf);
        }
        else
#endif
        {
            L2C_LINK_SEND_ACL_DATA (p_buf);//发送数据包
        }
    }

#if (L2CAP_HCI_FLOW_CONTROL_DEBUG == TRUE)
#if (BLE_INCLUDED == TRUE)
    if (p_lcb->is_ble_link)
    {
        L2CAP_TRACE_DEBUG6 ("TotalWin=%d,Hndl=0x%x,Quota=%d,Unack=%d,RRQuota=%d,RRUnack=%d",
                l2cb.controller_le_xmit_window,
                p_lcb->handle,
                p_lcb->link_xmit_quota, p_lcb->sent_not_acked,
                l2cb.round_robin_quota, l2cb.round_robin_unacked);
    }
    else
#endif
    {
        L2CAP_TRACE_DEBUG6 ("TotalWin=%d,Hndl=0x%x,Quota=%d,Unack=%d,RRQuota=%d,RRUnack=%d",
                l2cb.controller_xmit_window,
                p_lcb->handle,
                p_lcb->link_xmit_quota, p_lcb->sent_not_acked,
                l2cb.round_robin_quota, l2cb.round_robin_unacked);
    }
#endif

    return TRUE;
}

{% endhighlight %}

l2c_link_send_to_lower 把数据交给了 L2C_LINK_SEND_ACL_DATA，L2C_LINK_SEND_ACL_DATA 其实是 bte_main_hci_send 函数，bte_main_hci_send 函数通过调用 hci 的接口 transmit_buf 来转送数据包。
transmit_buf 函数作用比较简单，将此数据包入 tx_q 队列，串口(H5)的守护线程 bt_hc_worker_thread 会从 tx_q 队列中获取数据包，并将其发送。

{% highlight c linenos %}
static int transmit_buf(TRANSAC transac, char *p_buf, int len)
{
    utils_enqueue(&tx_q, (void *) transac);

    bthc_signal_event(HC_EVENT_TX);

    return BT_HC_STATUS_SUCCESS;
}
{% endhighlight %}

到这里为止，我们就把 ACL 包整个发送流程分析完了。
