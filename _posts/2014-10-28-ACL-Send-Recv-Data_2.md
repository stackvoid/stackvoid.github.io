---
layout: post
title: bluedroid源码分析之ACL包发送和接收（二）
categories: [bluedroid]
tags: [ACL]
---

上一节讲了数据流入口，本文分析L2CAP的处理函数。

###L2CAP层的处理

我们的音乐数据，通过 L2CAP 入口函数 l2c_data_write 的层层“考验”，已经顺利进入到 L2CAP 里了，下面我们来看看 L2CAP 层具体是怎么处理数据的。

首先我们进入了 L2CAP 层的状态机。

{% highlight c linenos %}
void l2c_csm_execute (tL2C_CCB *p_ccb, UINT16 event, void *p_data)
{
    switch (p_ccb->chnl_state)
    {
    case CST_CLOSED:
        l2c_csm_closed (p_ccb, event, p_data);
        break;

    case CST_ORIG_W4_SEC_COMP:
        l2c_csm_orig_w4_sec_comp (p_ccb, event, p_data);
        break;

    case CST_TERM_W4_SEC_COMP:
        l2c_csm_term_w4_sec_comp (p_ccb, event, p_data);
        break;

    case CST_W4_L2CAP_CONNECT_RSP:
        l2c_csm_w4_l2cap_connect_rsp (p_ccb, event, p_data);
        break;

    case CST_W4_L2CA_CONNECT_RSP:
        l2c_csm_w4_l2ca_connect_rsp (p_ccb, event, p_data);
        break;

    case CST_CONFIG:
        l2c_csm_config (p_ccb, event, p_data);
        break;

    case CST_OPEN:
        l2c_csm_open (p_ccb, event, p_data);
        break;

    case CST_W4_L2CAP_DISCONNECT_RSP:
        l2c_csm_w4_l2cap_disconnect_rsp (p_ccb, event, p_data);
        break;

    case CST_W4_L2CA_DISCONNECT_RSP:
        l2c_csm_w4_l2ca_disconnect_rsp (p_ccb, event, p_data);
        break;

    default:
        break;
    }
}
{% endhighlight %}

具体的 Channel 状态信息如下

{% highlight c linenos %}
typedef enum
{
    CST_CLOSED,                           /* Channel is in clodes state           */
    CST_ORIG_W4_SEC_COMP,                 /* Originator waits security clearence  */
    CST_TERM_W4_SEC_COMP,                 /* Acceptor waits security clearence    */
    CST_W4_L2CAP_CONNECT_RSP,             /* Waiting for peer conenct response    */
    CST_W4_L2CA_CONNECT_RSP,              /* Waiting for upper layer connect rsp  */
    CST_CONFIG,                           /* Negotiating configuration            */
    CST_OPEN,                             /* Data transfer state                  */
    CST_W4_L2CAP_DISCONNECT_RSP,          /* Waiting for peer disconnect rsp      */
    CST_W4_L2CA_DISCONNECT_RSP            /* Waiting for upper layer disc rsp     */
} tL2C_CHNL_STATE;
{% endhighlight %}

**l2c_csm_execute** 函数通过 p_ccb 中的 chnl_state 字段，来确定接下来数据包的走向。如下图所示：

![ACL Data](/album/l2cap_state_machine.png)

1. CST_CLOSED 状态：这个 case 处理 Channel 处于 CLOSED 状态的事件。这个状态仅仅存在于 L2CAP 的 Link 初始建立过程中。

- 如果发现事件是 L2CEVT_LP_DISCONNECT_IND，则当前 Link 已经断开，则释放当前 Channel的 ccb；
- 若事件是 L2CEVT_LP_CONNECT_CFM，则置 p_ccb->chnl_state 为  CST_ORIG_W4_SEC_COMP 状态，下面会接着介绍这个。
- 如果是 L2CEVT_LP_CONNECT_CFM_NEG 则说明当前 Link 失败，
- 如果是 L2CEVT_SEC_COMP 则说明 Security 已经清除成功。
- 若是 L2CEVT_L2CA_CONNECT_REQ 则说明 来自上层的 connect 请求，如果当前处于 sniff 状态，要先取消 sniff。
- L2CEVT_SEC_COMP_NEG 说明 Security 失败，清除当前 CCB，返回
- L2CEVT_L2CAP_CONNECT_REQ 说明是 Peer connect request，既然成功连接了，结束掉 timer
- L2CEVT_L2CA_DATA_WRITE，如果我们的数据从上层经过这里，并且是 CST_CLOSED，由于当前的 Channel 没有建立好，并且上层已经将数据丢给L2CAP了，只能将数据丢弃处理了(数据流不能逆向)。
- L2CEVT_L2CA_DISCONNECT_REQ，上层想断开链接，会使用这个 Event来处理。

2. CST_ORIG_W4_SEC_COMP 状态：Originator(我的理解是 发起 link 建立的应用)等待 security 的间隙，这个间隙需要处理的事件。跟 CST_CLOSED 差不多，源码很容易读，这里不再次做分析。注意，如果是 L2CEVT_SEC_COMP 事件（跟安全相关），会把 p_ccb->chnl_state 置为 CST_W4_L2CAP_CONNECT_RSP
3. CST_TERM_W4_SEC_COMP状态：Acceptor(接收者)等待 security 的间隙，源码易读，不再详细展开分析，注意一个事件 L2CEVT_SEC_COMP 的处理，会将 p_ccb->chnl_state 置为 CST_W4_L2CA_CONNECT_RSP
4. CST_W4_L2CAP_CONNECT_RSP： 经过 2 或 3 这个关于安全链接的步骤，当然要等待 Peer connect的回应(Response)。分为 获取 peer connect的 confirm、pending、rejected connection等信息，作进一步的判断。
5. CST_CONFIG：商讨配置的过程。
6. CST_OPEN：Channel 处于 OPEN 状态，我们可以发送上层传过来的数据，我们的音乐数据就是从这个 case 里发出去的。
7. CST_W4_L2CAP_DISCONNECT_RSP：等待 peer(对方设备)断开链接的 Response
8. CST_W4_L2CA_DISCONNECT_RSP：等待上层 disc rsp

分析了这么一大堆，我们的听音乐那那笔数据包，走的是 CST_OPEN 这个case，因为别的几个 case 在link 建立完成时就走完了。

我们的音乐数据包走的是 CST_OPEN 这个 case，这个 case 调用的是 l2c_csm_open 这个函数，接下来我们继续分析这个函数。

我们的音乐数据包在函数 l2c_csm_open 中流转，经过各种选择和判断，最后走的是  L2CEVT_L2CA_DATA_WRITE 这个 case。这个 case 调用了 l2c_enqueue_peer_data 让数据进入到当前 ccb 的 xmit_hold_q 队列中，暂存此数据包。l2c_link_check_send_pkts 这个函数发送数据包。下面我们会继续分析这两个函数。

{% highlight c linenos %}
//l2c_csm_open 处理 Channel 处于 OPEN 状态下的各种 Event
static void l2c_csm_open (tL2C_CCB *p_ccb, UINT16 event, void *p_data)
{
    UINT16                  local_cid = p_ccb->local_cid;
    tL2CAP_CFG_INFO         *p_cfg;
    tL2C_CHNL_STATE         tempstate;
    UINT8                   tempcfgdone;
    UINT8                   cfg_result;

#if (BT_TRACE_VERBOSE == TRUE)
    L2CAP_TRACE_EVENT2 ("L2CAP - LCID: 0x%04x  st: OPEN  evt: %s", p_ccb->local_cid, l2c_csm_get_event_name (event));
#else
    L2CAP_TRACE_EVENT1 ("L2CAP - st: OPEN evt: %d", event);
#endif

#if (L2CAP_UCD_INCLUDED == TRUE) //默认 UCD 是关闭的
    if ( local_cid == L2CAP_CONNECTIONLESS_CID )
    {
        /* check if this event can be processed by UCD */
        if ( l2c_ucd_process_event (p_ccb, event, p_data) )
        {
            /* The event is processed by UCD state machine */
            return;
        }
    }
#endif

    switch (event)
    {
    case L2CEVT_LP_DISCONNECT_IND:  //Link 都断开连接了，自然 Channel也没有存在的必要了，各种清除 CCB 的工作
        L2CAP_TRACE_API1 ("L2CAP - Calling Disconnect_Ind_Cb(), CID: 0x%04x  No Conf Needed", p_ccb->local_cid);
        l2cu_release_ccb (p_ccb);//释放 当前的 CCB 
        if (p_ccb->p_rcb)
            (*p_ccb->p_rcb->api.pL2CA_DisconnectInd_Cb)(local_cid, FALSE);
        break;

    case L2CEVT_LP_QOS_VIOLATION_IND:               /* QOS violation         */
        /* Tell upper layer. If service guaranteed, then clear the channel   */
        if (p_ccb->p_rcb->api.pL2CA_QoSViolationInd_Cb)
            (*p_ccb->p_rcb->api.pL2CA_QoSViolationInd_Cb)(p_ccb->p_lcb->remote_bd_addr);
        break;

    case L2CEVT_L2CAP_CONFIG_REQ:                  /* Peer config request   */
        p_cfg = (tL2CAP_CFG_INFO *)p_data;

        tempstate = p_ccb->chnl_state;
        tempcfgdone = p_ccb->config_done;
        p_ccb->chnl_state = CST_CONFIG; //如果数据流中的数据是 L2CEVT_L2CAP_CONFIG_REQ，当然要转到 CST_CONFIG中继续处理
        p_ccb->config_done &= ~CFG_DONE_MASK;
		//启动一个 timer ，一段时间后，查看 cfg 的状态
		//如果配置处于 L2CAP_PEER_CFG_UNACCEPTABLE，继续尝试配置
		//如果配置处于断开状态，那当前 Channel 直接断开连接。
        btu_start_timer (&p_ccb->timer_entry, BTU_TTYPE_L2CAP_CHNL, L2CAP_CHNL_CFG_TIMEOUT);

        if ((cfg_result = l2cu_process_peer_cfg_req (p_ccb, p_cfg)) == L2CAP_PEER_CFG_OK)
        {
            (*p_ccb->p_rcb->api.pL2CA_ConfigInd_Cb)(p_ccb->local_cid, p_cfg);
        }

        /* Error in config parameters: reset state and config flag */
        else if (cfg_result == L2CAP_PEER_CFG_UNACCEPTABLE)
        {
            btu_stop_timer(&p_ccb->timer_entry);
            p_ccb->chnl_state = tempstate;
            p_ccb->config_done = tempcfgdone;
            l2cu_send_peer_config_rsp (p_ccb, p_cfg);
        }
        else    /* L2CAP_PEER_CFG_DISCONNECT */
        {
            /* Disconnect if channels are incompatible
             * Note this should not occur if reconfigure
             * since this should have never passed original config.
             */
            l2cu_disconnect_chnl (p_ccb);
        }
        break;

    case L2CEVT_L2CAP_DISCONNECT_REQ:                  /* Peer disconnected request */
// btla-specific ++
        /* Make sure we are not in sniff mode */
#if BTM_PWR_MGR_INCLUDED == TRUE
        {
            tBTM_PM_PWR_MD settings;
            settings.mode = BTM_PM_MD_ACTIVE;
            BTM_SetPowerMode (BTM_PM_SET_ONLY_ID, p_ccb->p_lcb->remote_bd_addr, &settings);
        }
#else
        BTM_CancelSniffMode (p_ccb->p_lcb->remote_bd_addr);
#endif
// btla-specific --

        p_ccb->chnl_state = CST_W4_L2CA_DISCONNECT_RSP; //Peer 发送 Disconnect，我们要对此发 Response
        btu_start_timer (&p_ccb->timer_entry, BTU_TTYPE_L2CAP_CHNL, L2CAP_CHNL_DISCONNECT_TOUT);
        L2CAP_TRACE_API1 ("L2CAP - Calling Disconnect_Ind_Cb(), CID: 0x%04x  Conf Needed", p_ccb->local_cid);
        (*p_ccb->p_rcb->api.pL2CA_DisconnectInd_Cb)(p_ccb->local_cid, TRUE);
        break;

    case L2CEVT_L2CAP_DATA:                         /* Peer data packet rcvd    */
		//收到 Peer 传来的数据，当然要把这个数据通过回调送到上层应用去
		//pL2CA_DataInd_Cb 中定义了回调，交给上层处理收到的数据
        (*p_ccb->p_rcb->api.pL2CA_DataInd_Cb)(p_ccb->local_cid, (BT_HDR *)p_data);
        break;

    case L2CEVT_L2CA_DISCONNECT_REQ:                 /* Upper wants to disconnect */
        /* Make sure we are not in sniff mode */
#if BTM_PWR_MGR_INCLUDED == TRUE
        {
            tBTM_PM_PWR_MD settings;
            settings.mode = BTM_PM_MD_ACTIVE;
            BTM_SetPowerMode (BTM_PM_SET_ONLY_ID, p_ccb->p_lcb->remote_bd_addr, &settings);
        }
#else
        BTM_CancelSniffMode (p_ccb->p_lcb->remote_bd_addr);
#endif

        l2cu_send_peer_disc_req (p_ccb);
        p_ccb->chnl_state = CST_W4_L2CAP_DISCONNECT_RSP;
        btu_start_timer (&p_ccb->timer_entry, BTU_TTYPE_L2CAP_CHNL, L2CAP_CHNL_DISCONNECT_TOUT);
        break;

    case L2CEVT_L2CA_DATA_WRITE:                    /* Upper layer data to send */   //mike mark l2c
		//上层将数据发送给下层
		//我们的音乐数据就是走这个 case(为什么？看整个函数的参数就明白了)
		//首先将数据入队，下面会展开分析这个函数
		l2c_enqueue_peer_data (p_ccb, (BT_HDR *)p_data);
		//最终调用 l2c_link_check_send_pkts 来发送我们的音乐数据包
        l2c_link_check_send_pkts (p_ccb->p_lcb, NULL, NULL);
        break;

    case L2CEVT_L2CA_CONFIG_REQ:                   /* Upper layer config req   */
        p_ccb->chnl_state = CST_CONFIG;
        p_ccb->config_done &= ~CFG_DONE_MASK;
        l2cu_process_our_cfg_req (p_ccb, (tL2CAP_CFG_INFO *)p_data);
        l2cu_send_peer_config_req (p_ccb, (tL2CAP_CFG_INFO *)p_data);
        btu_start_timer (&p_ccb->timer_entry, BTU_TTYPE_L2CAP_CHNL, L2CAP_CHNL_CFG_TIMEOUT);
        break;

    case L2CEVT_TIMEOUT:
        /* Process the monitor/retransmission time-outs in flow control/retrans mode */
        if (p_ccb->peer_cfg.fcr.mode == L2CAP_FCR_ERTM_MODE)
            l2c_fcr_proc_tout (p_ccb);
        break;

    case L2CEVT_ACK_TIMEOUT:
        l2c_fcr_proc_ack_tout (p_ccb);
        break;
    }
}
{% endhighlight %}

OK，我们下篇将分析数据包入队列的函数。
