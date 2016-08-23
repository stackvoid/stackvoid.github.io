---
layout: post
title: bluedroid源码分析之ACL包发送和接收（五）
categories: [bluedroid]
tags: [ACL]
---

### ACL 包接收流程

有关 ACL 包接收的过程都是在 btu_task 这个守护线程中处理的。

我们看到 **btu_task** 处理数据包的过程：

1. 等待事件，事件到来后，如果是 TASK_MBOX_0_EVT_MASK(是不是 MBOX里的Task)，那么从 mbox 中取出这个数据包，并判断是什么类型的 Event。
2. 如果是 BT_EVT_TO_BTU_HCI_ACL，说明是 ACL 数据，交给 l2cap 来处理。
3. 如果是 BT_EVT_TO_BTU_L2C_SEG_XMIT,说明是 L2CAP 的分包数据没有发送完，那继续发送分包数据。

{% highlight c linenos %}
//部分 btu_task 源码
...........
   /* Wait for, and process, events */
    for (;;)
    {
        event = GKI_wait (0xFFFF, 0);

        if (event & TASK_MBOX_0_EVT_MASK)
        {
            /* Process all messages in the queue */
            while ((p_msg = (BT_HDR *) GKI_read_mbox (BTU_HCI_RCV_MBOX)) != NULL)
            {
                /* Determine the input message type. */
                switch (p_msg->event & BT_EVT_MASK)
                {
                    case BT_EVT_TO_BTU_HCI_ACL:
                        /* All Acl Data goes to L2CAP */
                        l2c_rcv_acl_data (p_msg);//我们的 ACL 数据来了，关键分析这个函数
                        break;

                    case BT_EVT_TO_BTU_L2C_SEG_XMIT:
                        /* L2CAP segment transmit complete */
                        l2c_link_segments_xmitted (p_msg);
                        break;

                    case BT_EVT_TO_BTU_HCI_SCO:
#if BTM_SCO_INCLUDED == TRUE
                        btm_route_sco_data (p_msg);
                        break;
#endif

                    case BT_EVT_TO_BTU_HCI_EVT:
                        btu_hcif_process_event ((UINT8)(p_msg->event & BT_SUB_EVT_MASK), p_msg);
                        GKI_freebuf(p_msg);

....

{% endhighlight %}

**l2c_rcv_acl_data** 这个函数，处理收到的 ACL 包，下面我们来分析一下 l2c_rcv_acl_data 这个函数：

1. 在收到的 ACL 包中找出 pkt_type(分包的话要另作处理) 和 handle。

2. 若此 ACL 包是一个完整的数据包：
   - 首先通过 handle 找到 LCB
   - rcv_cid 大于 L2CAP_BASE_APPL_CID(0x0040),说明是上层应用普通数据包，通过 CID 找到当前包的 CCB。
   - hci_len 长度肯定要大于 L2CAP 头长度，否则肯定头部出错了。
   - 如果 rcv_cid 是 L2CAP_SIGNALLING_CID，说明数据包是 创建和建立 Channel 用的(上层应用传输数据)，使用函数  process_l2cap_cmd 来处理。
   - 如果 rcv_cid 是 L2CAP_CONNECTIONLESS_CID 说明是 广播或单播，使用函数 tcs_proc_bcst_msg 处理。
   - 如果 rcv_cid 是 L2CAP_BLE_SIGNALLING_CID 说明是 BLE 的signalling包，交给函数 l2cble_process_sig_cmd 处理。
   - 普通数据包，直接交给 L2CAP 的数据流状态机 l2c_csm_execute (p_ccb, **L2CEVT_L2CAP_DATA**, p_msg) 来处理，找到 L2CEVT_L2CAP_DATA 的这个 case，原来通过回调，将此数据包交给上层了(如 RFCOMM，最终是交给 RFCOMM_BufDataInd函数作进一步处理)

上述数据包，我们仅仅考虑了最普通的数据流，其他的控制数据包有时间在分析。

{% highlight c linenos %}
void l2c_rcv_acl_data (BT_HDR *p_msg)
{
    UINT8       *p = (UINT8 *)(p_msg + 1) + p_msg->offset;
    UINT16      handle, hci_len;
    UINT8       pkt_type;
    tL2C_LCB    *p_lcb;
    tL2C_CCB    *p_ccb = NULL;
    UINT16      l2cap_len, rcv_cid, psm;

    /* Extract the handle */
    STREAM_TO_UINT16 (handle, p);
    pkt_type = HCID_GET_EVENT (handle);
    handle   = HCID_GET_HANDLE (handle);

    /* Since the HCI Transport is putting segmented packets back together, we */
    /* should never get a valid packet with the type set to "continuation"    */
    if (pkt_type != L2CAP_PKT_CONTINUE)//数据包一定要是完整的，分包另作处理
    {
        /* Find the LCB based on the handle */
        if ((p_lcb = l2cu_find_lcb_by_handle (handle)) == NULL)
        {
            UINT8       cmd_code;

            /* There is a slight possibility (specifically with USB) that we get an */
            /* L2CAP connection request before we get the HCI connection complete.  */
            /* So for these types of messages, hold them for up to 2 seconds.       */
            STREAM_TO_UINT16 (hci_len, p);
            STREAM_TO_UINT16 (l2cap_len, p);
            STREAM_TO_UINT16 (rcv_cid, p);
            STREAM_TO_UINT8  (cmd_code, p);

            if ((p_msg->layer_specific == 0) && (rcv_cid == L2CAP_SIGNALLING_CID)
                && (cmd_code == L2CAP_CMD_INFO_REQ || cmd_code == L2CAP_CMD_CONN_REQ))
            {
                L2CAP_TRACE_WARNING5 ("L2CAP - holding ACL for unknown handle:%d ls:%d cid:%d opcode:%d cur count:%d",
                                    handle, p_msg->layer_specific, rcv_cid, cmd_code,
                                    l2cb.rcv_hold_q.count);
                p_msg->layer_specific = 2;
                GKI_enqueue (&l2cb.rcv_hold_q, p_msg);//添加到队列中，等待 connect 数据包到达，有种可能发生的 case

                if (l2cb.rcv_hold_q.count == 1)
                    btu_start_timer (&l2cb.rcv_hold_tle, BTU_TTYPE_L2CAP_HOLD, BT_1SEC_TIMEOUT);

                return;
            }
            else
            {
                L2CAP_TRACE_ERROR5 ("L2CAP - rcvd ACL for unknown handle:%d ls:%d cid:%d opcode:%d cur count:%d",
                                    handle, p_msg->layer_specific, rcv_cid, cmd_code, l2cb.rcv_hold_q.count);
            }
            GKI_freebuf (p_msg);
            return;
        }
    }
    else
    {
        L2CAP_TRACE_WARNING1 ("L2CAP - expected pkt start or complete, got: %d", pkt_type);
        GKI_freebuf (p_msg);
        return;
    }
    //下面是我们把 ACL 数据包给部分拆包了，即除掉 L2CAP 的控制部分，还原上层的数据包。
    /* Extract the length and update the buffer header */
    STREAM_TO_UINT16 (hci_len, p);
    p_msg->offset += 4;

#if (L2CAP_HOST_FLOW_CTRL == TRUE)
    /* Send ack if we hit the threshold */
    if (++p_lcb->link_pkts_unacked >= p_lcb->link_ack_thresh)
        btu_hcif_send_host_rdy_for_data();
#endif

    /* Extract the length and CID */
    STREAM_TO_UINT16 (l2cap_len, p);
    STREAM_TO_UINT16 (rcv_cid, p);

    /* Find the CCB for this CID */
    if (rcv_cid >= L2CAP_BASE_APPL_CID)// 说明此 rcv_cid 是上层应用普通数据流的 CID 
    {
        if ((p_ccb = l2cu_find_ccb_by_cid (p_lcb, rcv_cid)) == NULL)
        {
            L2CAP_TRACE_WARNING1 ("L2CAP - unknown CID: 0x%04x", rcv_cid);
            GKI_freebuf (p_msg);
            return;
        }
    }

    if (hci_len >= L2CAP_PKT_OVERHEAD)  //数据包长度肯定要大于 Head的值，否则必然是个错包
    {
        p_msg->len    = hci_len - L2CAP_PKT_OVERHEAD;
        p_msg->offset += L2CAP_PKT_OVERHEAD;
    }
    else
    {
        L2CAP_TRACE_WARNING0 ("L2CAP - got incorrect hci header" );
        GKI_freebuf (p_msg);
        return;
    }

    if (l2cap_len != p_msg->len) //长度不相等，那数据包传送过程肯定出现了差错，丢包吧
    {
        L2CAP_TRACE_WARNING2 ("L2CAP - bad length in pkt. Exp: %d  Act: %d",
                              l2cap_len, p_msg->len);

        GKI_freebuf (p_msg);
        return;
    }

    /* Send the data through the channel state machine */
    if (rcv_cid == L2CAP_SIGNALLING_CID)//控制创建和建立 Channel的 signalling
    {
        process_l2cap_cmd (p_lcb, p, l2cap_len); //此函数专门处理这个Channel的事件
        GKI_freebuf (p_msg);
    }
    else if (rcv_cid == L2CAP_CONNECTIONLESS_CID)
    {
        /* process_connectionless_data (p_lcb); */
        STREAM_TO_UINT16 (psm, p);
        L2CAP_TRACE_DEBUG1( "GOT CONNECTIONLESS DATA PSM:%d", psm ) ;
#if (TCS_BCST_SETUP_INCLUDED == TRUE && TCS_INCLUDED == TRUE)
        if (psm == TCS_PSM_INTERCOM || psm == TCS_PSM_CORDLESS)
        {
            p_msg->offset += L2CAP_BCST_OVERHEAD;
            p_msg->len -= L2CAP_BCST_OVERHEAD;
            tcs_proc_bcst_msg( p_lcb->remote_bd_addr, p_msg ) ;
            GKI_freebuf (p_msg);
        }
        else
#endif

#if (L2CAP_UCD_INCLUDED == TRUE)
        /* if it is not broadcast, check UCD registration */
        if ( l2c_ucd_check_rx_pkts( p_lcb, p_msg ) )
        {
            /* nothing to do */
        }
        else
#endif
            GKI_freebuf (p_msg);
    }
#if (BLE_INCLUDED == TRUE)
    else if (rcv_cid == L2CAP_BLE_SIGNALLING_CID) //LE 设备的专用 Channel
    {
        l2cble_process_sig_cmd (p_lcb, p, l2cap_len);
        GKI_freebuf (p_msg);
    }
#endif
#if (L2CAP_NUM_FIXED_CHNLS > 0)
    else if ((rcv_cid >= L2CAP_FIRST_FIXED_CHNL) && (rcv_cid <= L2CAP_LAST_FIXED_CHNL) &&
             (l2cb.fixed_reg[rcv_cid - L2CAP_FIRST_FIXED_CHNL].pL2CA_FixedData_Cb != NULL) )
    {
        /* If no CCB for this channel, allocate one */
        if (l2cu_initialize_fixed_ccb (p_lcb, rcv_cid, &l2cb.fixed_reg[rcv_cid - L2CAP_FIRST_FIXED_CHNL].fixed_chnl_opts))
        {
            p_ccb = p_lcb->p_fixed_ccbs[rcv_cid - L2CAP_FIRST_FIXED_CHNL];

            if (p_ccb->peer_cfg.fcr.mode != L2CAP_FCR_BASIC_MODE)
                l2c_fcr_proc_pdu (p_ccb, p_msg);
            else
                (*l2cb.fixed_reg[rcv_cid - L2CAP_FIRST_FIXED_CHNL].pL2CA_FixedData_Cb)(p_lcb->remote_bd_addr, p_msg);
        }
        else
            GKI_freebuf (p_msg);
    }
#endif

    else
    {
        if (p_ccb == NULL)
            GKI_freebuf (p_msg);
        else
        {
            /* Basic mode packets go straight to the state machine */
            if (p_ccb->peer_cfg.fcr.mode == L2CAP_FCR_BASIC_MODE)
			//普通的数据流都是经过这条通路的，下面这个函数觉得很熟悉吧，因为在发送数据包的时候也调用了他。
                l2c_csm_execute (p_ccb, L2CEVT_L2CAP_DATA, p_msg);
            else
            {
                /* eRTM or streaming mode, so we need to validate states first */
                if ((p_ccb->chnl_state == CST_OPEN) || (p_ccb->chnl_state == CST_CONFIG))
                    l2c_fcr_proc_pdu (p_ccb, p_msg);
                else
                    GKI_freebuf (p_msg);
            }
        }
    }
}
{% endhighlight %}

下面我们分析一个 Event--number-of-completed-packets，是由函数 l2c_link_process_num_completed_pkts 处理。
这个 event 将会更新我们 LCB 上的数据包数量。

我们追踪这个函数的调用链：

btu_task的一个 BT_EVT_TO_BTU_HCI_EVT case --> btu_hcif_process_event 的一个 HCI_NUM_COMPL_DATA_PKTS_EVT case -->  l2c_link_process_num_completed_pkts

通过这个调用链，我们知道处理事件或数据流都在 btu_task 中进行。

**l2c_link_process_num_completed_pkts**这个函数都做了些什么呢？我们来深入代码了解一下整个过程。

1. 通过 num_sent 来更新数据 controller_xmit_window，sent_not_acked。
2. 既然又多了 credits(Snoop中的)，那么自然调用 l2c_link_check_send_pkts 去找更多的包发送下去。
3. l2c_link_process_num_completed_pkts 和 l2c_link_check_send_pkts 形成了一个递归式的调用，l2c_link_check_send_pkts 会产生 process num complete 这个event，l2c_link_process_num_completed_pkts找到 Link后在让 l2c_link_check_send_pkts 发包。

{% highlight c linenos %}
void l2c_link_process_num_completed_pkts (UINT8 *p)
{
    UINT8       num_handles, xx;
    UINT16      handle;
    UINT16      num_sent; //已经发下去的数据，这个数据要更新controller_xmit_window
    tL2C_LCB    *p_lcb;

    L2CAP_TRACE_DEBUG0("mike: l2c_link_process_num_completed_pkts");
    STREAM_TO_UINT8 (num_handles, p);
    L2CAP_TRACE_DEBUG1("mike: l2c_link_process_num_completed_pkts--number_handles:%d", num_handles);
    for (xx = 0; xx < num_handles; xx++)//handle 对应着每条逻辑链路(Link)
    {
        STREAM_TO_UINT16 (handle, p);
        STREAM_TO_UINT16 (num_sent, p);

        p_lcb = l2cu_find_lcb_by_handle (handle);

        /* Callback for number of completed packet event    */
        /* Originally designed for [3DSG]                   */
        if((p_lcb != NULL) && (p_lcb->p_nocp_cb))
        {
            L2CAP_TRACE_DEBUG0 ("L2CAP - calling NoCP callback");
            (*p_lcb->p_nocp_cb)(p_lcb->remote_bd_addr);
        }

        if (p_lcb)
        {
#if (BLE_INCLUDED == TRUE)
            if (p_lcb->is_ble_link)
            {
                l2cb.controller_le_xmit_window += num_sent;
            }
            else
#endif
            {
                /* Maintain the total window to the controller */
                l2cb.controller_xmit_window += num_sent;
            }
            /* If doing round-robin, adjust communal counts */
            if (p_lcb->link_xmit_quota == 0)
            {
                /* Don't go negative */
                if (l2cb.round_robin_unacked > num_sent)
                    l2cb.round_robin_unacked -= num_sent;
                else
                    l2cb.round_robin_unacked = 0;
            }

            /* Don't go negative */
            if (p_lcb->sent_not_acked > num_sent)
                p_lcb->sent_not_acked -= num_sent; //更新
            else
                p_lcb->sent_not_acked = 0;

            l2c_link_check_send_pkts (p_lcb, NULL, NULL);

			L2CAP_TRACE_DEBUG1("mike:l2c_link_process_num_completed_pkts--l2cb.controller_xmit_window=%d",l2cb.controller_xmit_window);

            /* If we were doing round-robin for low priority links, check 'em */
            if ( (p_lcb->acl_priority == L2CAP_PRIORITY_HIGH)
              && (l2cb.check_round_robin)
              && (l2cb.round_robin_unacked < l2cb.round_robin_quota) )
            {
              l2c_link_check_send_pkts (NULL, NULL, NULL);
            }
        }

#if (L2CAP_HCI_FLOW_CONTROL_DEBUG == TRUE)
        if (p_lcb)
        {
#if (BLE_INCLUDED == TRUE)
            if (p_lcb->is_ble_link)
            {
                L2CAP_TRACE_DEBUG5 ("TotalWin=%d,LinkUnack(0x%x)=%d,RRCheck=%d,RRUnack=%d",
                    l2cb.controller_le_xmit_window,
                    p_lcb->handle, p_lcb->sent_not_acked,
                    l2cb.check_round_robin, l2cb.round_robin_unacked);
            }
            else
#endif
            {
                L2CAP_TRACE_DEBUG5 ("TotalWin=%d,LinkUnack(0x%x)=%d,RRCheck=%d,RRUnack=%d",
                    l2cb.controller_xmit_window,
                    p_lcb->handle, p_lcb->sent_not_acked,
                    l2cb.check_round_robin, l2cb.round_robin_unacked);

            }
        }
        else
        {
#if (BLE_INCLUDED == TRUE)
            L2CAP_TRACE_DEBUG5 ("TotalWin=%d  LE_Win: %d, Handle=0x%x, RRCheck=%d, RRUnack=%d",
                l2cb.controller_xmit_window,
                l2cb.controller_le_xmit_window,
                handle,
                l2cb.check_round_robin, l2cb.round_robin_unacked);
#else
            L2CAP_TRACE_DEBUG4 ("TotalWin=%d  Handle=0x%x  RRCheck=%d  RRUnack=%d",
                l2cb.controller_xmit_window,
                handle,
                l2cb.check_round_robin, l2cb.round_robin_unacked);
#endif
        }
#endif
    }

#if (defined(HCILP_INCLUDED) && HCILP_INCLUDED == TRUE)
    /* only full stack can enable sleep mode */
    btu_check_bt_sleep ();
#endif
}

{% endhighlight %}

到此为止，我们已经把普通 ACL 包发送和接收部分源码分析完了。后续有时间会继续分析控制类型的ACL包，以及ACL链路建立的流程。
