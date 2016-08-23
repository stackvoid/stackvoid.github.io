---
layout: post
title: bluedroid源码分析之ACL包发送和接收（三）
categories: [bluedroid]
tags: [ACL]
---

本文讲的是 Bluedroid 中数据包在 L2CAP 层入队列的一系列函数源码分析。

**l2c_enqueue_peer_data** 函数的主要作用是将我们的音乐数据包入数据发送队列以及处理 FCR segmentation 和当前 Channel 是否拥堵的检测，我们来详细读一下他的源码。其主要做了这么几件事：
1. 组装好 p_buf 并入 当前 CCB 的 xmit_hold_q 队列。
2. 检查当前 Channel 拥堵情况。
3. 当前 Link 支持 RR，则检查当前ACL数据包所在 Channel 的权限，如果当前 CCB 中的权限高于 RR，则把 RR 中的权限设置为跟 CCB 相同。
4. 若 Link 上没有发送窗口，则将 l2cb.check_round_robin 置为TRUE，下一次需要 RR 

{% highlight c linenos %}
void l2c_enqueue_peer_data (tL2C_CCB *p_ccb, BT_HDR *p_buf)
{
    UINT8       *p;

    if (p_ccb->peer_cfg.fcr.mode != L2CAP_FCR_BASIC_MODE)
    {
        p_buf->event = 0;
    }
    else
    {
        /* Save the channel ID for faster counting */
        p_buf->event = p_ccb->local_cid;

        /* Step back to add the L2CAP header */
        p_buf->offset -= L2CAP_PKT_OVERHEAD;
        p_buf->len    += L2CAP_PKT_OVERHEAD;

        /* Set the pointer to the beginning of the data */
        p = (UINT8 *)(p_buf + 1) + p_buf->offset;

        /* Now the L2CAP header */
        UINT16_TO_STREAM (p, p_buf->len - L2CAP_PKT_OVERHEAD);
        UINT16_TO_STREAM (p, p_ccb->remote_cid);
    }

    GKI_enqueue (&p_ccb->xmit_hold_q, p_buf);//真正将组装好的 p_buf 入队

    l2cu_check_channel_congestion (p_ccb);  //检测当前 Channel 拥堵情况，下面会继续分析这个函数

#if (L2CAP_ROUND_ROBIN_CHANNEL_SERVICE == TRUE)
    /* if new packet is higher priority than serving ccb and it is not overrun */
    if (( p_ccb->p_lcb->rr_pri > p_ccb->ccb_priority ) //当前数据包所在Channel的权限
      &&( p_ccb->p_lcb->rr_serv[p_ccb->ccb_priority].quota > 0))
    {
        /* send out higher priority packet */
        p_ccb->p_lcb->rr_pri = p_ccb->ccb_priority;//当前要发送的数据的Channel，设置为ccb_priority，比原来权限要高。
    }
#endif

    //如果当前 link 上的 link_xmit_quota ==0（link上的发送窗口为0），那么有必要做一次 RR
    if (p_ccb->p_lcb->link_xmit_quota == 0)
        l2cb.check_round_robin = TRUE;
}

//check if any change in congestion status

void l2cu_check_channel_congestion (tL2C_CCB *p_ccb)
{
    UINT16 q_count = p_ccb->xmit_hold_q.count; //当前 CCB 中 发送数据队列中数据包的总数

#if (L2CAP_UCD_INCLUDED == TRUE)
    if ( p_ccb->local_cid == L2CAP_CONNECTIONLESS_CID )
    {
        q_count += p_ccb->p_lcb->ucd_out_sec_pending_q.count;
    }
#endif

    /* If the CCB queue limit is subject to a quota, check for congestion */

    /* if this channel has outgoing traffic */
    if ((p_ccb->p_rcb)&&(p_ccb->buff_quota != 0))
    {
        /* If this channel was congested */
        if ( p_ccb->cong_sent ) //当前 Channel 的这个字段为TRUE，是否真正拥堵，我们要继续判断
        {
            /* If the channel is not congested now, tell the app */
			//p_ccb->buff_quota = quota_per_weighted_chnls[HCI_ACL_POOL_ID] * p_ccb->tx_data_rate
			//在函数 l2c_link_adjust_chnl_allocation 中配置此值
            if (q_count <= (p_ccb->buff_quota / 2))//q_count为 CCB 中的xmit_hold_q
            {
                p_ccb->cong_sent = FALSE; //当前CCB中的 xmit_hold_q 小于 buffer_quota 值的一半，就认为已经不拥堵了
                if (p_ccb->p_rcb->api.pL2CA_CongestionStatus_Cb)
                {
                    L2CAP_TRACE_DEBUG3 ("L2CAP - Calling CongestionStatus_Cb (FALSE), CID: 0x%04x  xmit_hold_q.count: %u  buff_quota: %u",
                                      p_ccb->local_cid, q_count, p_ccb->buff_quota);

                    /* Prevent recursive calling */
                    l2cb.is_cong_cback_context = TRUE;
                    (*p_ccb->p_rcb->api.pL2CA_CongestionStatus_Cb)(p_ccb->local_cid, FALSE);
                    l2cb.is_cong_cback_context = FALSE;
                }
#if (L2CAP_UCD_INCLUDED == TRUE)
                else if ( p_ccb->local_cid == L2CAP_CONNECTIONLESS_CID )//无连接的 CID
                {
                    if ( p_ccb->p_rcb->ucd.cb_info.pL2CA_UCD_Congestion_Status_Cb )
                    {
                        L2CAP_TRACE_DEBUG3 ("L2CAP - Calling UCD CongestionStatus_Cb (FALSE), SecPendingQ:%u,XmitQ:%u,Quota:%u",
                                             p_ccb->p_lcb->ucd_out_sec_pending_q.count,
                                             p_ccb->xmit_hold_q.count, p_ccb->buff_quota);
                        p_ccb->p_rcb->ucd.cb_info.pL2CA_UCD_Congestion_Status_Cb( p_ccb->p_lcb->remote_bd_addr, FALSE );
                    }
                }
#endif
            }
        }
        else
        {
            /* If this channel was not congested but it is congested now, tell the app */
            if (q_count > p_ccb->buff_quota) //此时仍然处于拥堵状态
            {
                p_ccb->cong_sent = TRUE;
                if (p_ccb->p_rcb->api.pL2CA_CongestionStatus_Cb)
                {
                    L2CAP_TRACE_DEBUG3 ("L2CAP - Calling CongestionStatus_Cb (TRUE),CID:0x%04x,XmitQ:%u,Quota:%u",
                        p_ccb->local_cid, q_count, p_ccb->buff_quota);

                    (*p_ccb->p_rcb->api.pL2CA_CongestionStatus_Cb)(p_ccb->local_cid, TRUE);
                }
#if (L2CAP_UCD_INCLUDED == TRUE)
                else if ( p_ccb->local_cid == L2CAP_CONNECTIONLESS_CID )
                {
                    if ( p_ccb->p_rcb->ucd.cb_info.pL2CA_UCD_Congestion_Status_Cb )
                    {
                        L2CAP_TRACE_DEBUG3 ("L2CAP - Calling UCD CongestionStatus_Cb (TRUE), SecPendingQ:%u,XmitQ:%u,Quota:%u",
                                             p_ccb->p_lcb->ucd_out_sec_pending_q.count,
                                             p_ccb->xmit_hold_q.count, p_ccb->buff_quota);
                        p_ccb->p_rcb->ucd.cb_info.pL2CA_UCD_Congestion_Status_Cb( p_ccb->p_lcb->remote_bd_addr, TRUE );
                    }
                }
#endif
            }
        }
    }
}
{% endhighlight %}

