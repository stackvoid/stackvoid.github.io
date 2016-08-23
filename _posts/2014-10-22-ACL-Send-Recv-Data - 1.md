---
layout: post
title: bluedroid源码分析之ACL包发送和接收（一）
categories: [bluedroid]
tags: [ACL]
---

ACL 链路在 Bluetooth 中非常重要，一些重要的应用如 A2DP，  基于 RFCOMM 的应用，BNEP等都要建立 ACL 链路，发送/接收ACL 包。Mike 跟大家一起来分析 ACL 包发送/接收流程，以及涉及到的重要 command/event。

----

###ACL包发送

下面的图([点击大图](stackvoid/stackvoid.github.io/blob/master/album/ACL_send.png))是各种应用层使用 L2CAP 的 API：L2CA_DataWrite 发送数据流的过程，此API继续往下走，我仅分析了正常数据流的走向(暂时没有考虑别的情况)。

![ACL_01](/album/ACL_send.png)

----

###应用层数据到 L2CAP 的入口

我们假设一个听音乐的场景，Mike跟大家一起分析音乐数据流 AVDTP 以下层的传送。

在 AVDTP 中，所有的功能想发送 Data，必须调用 avdt_ad_write_req 这个函数，Mike 就从这个函数入手分析。

{% highlight c linenos%}
//当CCB或SCB给l2cap的 Channel 发送数据时，他们最终都会使用到L2CAP的 API：L2CA_Data_Write
UINT8 avdt_ad_write_req(UINT8 type, tAVDT_CCB *p_ccb, tAVDT_SCB *p_scb, BT_HDR *p_buf)
{
    UINT8   tcid;

    /* get tcid from type, scb */
    tcid = avdt_ad_type_to_tcid(type, p_scb);


    return L2CA_DataWrite(avdt_cb.ad.rt_tbl[avdt_ccb_to_idx(p_ccb)][tcid].lcid, p_buf);
}
//L2CA_DataWrite的返回形式有三种，分别是：
//1. L2CAP_DW_SUCCESS:此数据写成功
//2.L2CAP_DW_CONGESTED:写数据成功，但是当前信道拥堵
//3.L2CAP_DW_FAILED:写数据失败
UINT8 L2CA_DataWrite (UINT16 cid, BT_HDR *p_data)
{
    L2CAP_TRACE_API2 ("L2CA_DataWrite()  CID: 0x%04x  Len: %d", cid, p_data->len);
    return l2c_data_write (cid, p_data, L2CAP_FLUSHABLE_CH_BASED);
}
{% endhighlight %}
当我们的音乐数据流到达 l2c_data_write 这个函数时，标志数据流正式进入到L2CAP层。
我们在下面的源码中深入分析 l2c_data_write 这个函数。

**l2c_data_write** 这个函数做的事情主要有：

1. 根据参数 cid(Channel ID) 找到 对应的 ccb(Channel Control Block), 找不到返回 **L2CAP_DW_FAILED**
2. 如果测试者 打开 TESTER 这个宏，发送任意数据，当数据大小 大于 MTU 最大值，也会返回  **L2CAP_DW_FAILED**
3. 通过检查 p_ccb->cong_sent 字段，TRUE，则说明当前 Channel 已经拥挤，此时L2CAP的这个Channel不在接收数据，返回 **L2CAP_DW_FAILED**
4. 以上三个条件都通过，说明数据可发送，将数据通过 l2c_csm_execute 继续处理。进入 l2c_csm_execute 函数，标志着这笔数据已经成功交给 l2CAP 来处理，与上层已经没有关系了。
5. l2c_csm_execute 函数执行结束后，再次检查 p_ccb->cong_sent 字段，看看当前的 Channel 是否拥挤，如果拥挤则告诉上层 L2CAP_DW_CONGESTED，否则返回 L2CAP_DW_SUCCESS，表示数据已经成功发送。

{% highlight c linenos%}
//返回的数据跟上面的 L2CA_DataWrite 作用相同
UINT8 l2c_data_write (UINT16 cid, BT_HDR *p_data, UINT16 flags)
{
    tL2C_CCB        *p_ccb;

    //遍历l2cb.ccb_pool，通过Channel ID找到对应的Channel Control Block
	//l2cu_find_ccb_by_cid 见下面源码注释
    if ((p_ccb = l2cu_find_ccb_by_cid (NULL, cid)) == NULL)
    {
        L2CAP_TRACE_WARNING1 ("L2CAP - no CCB for L2CA_DataWrite, CID: %d", cid);
        GKI_freebuf (p_data);
        return (L2CAP_DW_FAILED);
    }

#ifndef TESTER /* Tester may send any amount of data. otherwise sending message
                  bigger than mtu size of peer is a violation of protocol */
    if (p_data->len > p_ccb->peer_cfg.mtu)
    {
        L2CAP_TRACE_WARNING1 ("L2CAP - CID: 0x%04x  cannot send message bigger than peer's mtu size", cid);
        GKI_freebuf (p_data);
        return (L2CAP_DW_FAILED);
    }
#endif

    /* channel based, packet based flushable or non-flushable */
    //Bluedroid中默认的是 L2CAP_FLUSHABLE_CH_BASED
    //这个 layer_specific 在 数据发送的 l2c_link_send_to_lower 中表示 ACL包分包 个数
    p_data->layer_specific = flags;

    //发现本 Channel 已经拥堵，直接返回L2CAP_DW_FAILED 告诉上层等会再发数据
	//当几个应用 共用 此 Channel 可能会出现这种情况
    if (p_ccb->cong_sent)
    {
        L2CAP_TRACE_ERROR3 ("L2CAP - CID: 0x%04x cannot send, already congested  xmit_hold_q.count: %u  buff_quota: %u",
                            p_ccb->local_cid, p_ccb->xmit_hold_q.count, p_ccb->buff_quota);

        GKI_freebuf (p_data);
        return (L2CAP_DW_FAILED);
    }
	//毫无疑问啦，这个函数就是我们继续需要分析的函数
    l2c_csm_execute (p_ccb, L2CEVT_L2CA_DATA_WRITE, p_data);

	//已经将上层的这笔数据发送完，如果此 Channel 拥挤了(之前发送这笔包还没拥挤)
	//返回 L2CAP_DW_CONGESTED 告诉上层当前信道拥挤，你要给我L2CAP层发数据，是不发下来的
    if (p_ccb->cong_sent)
        return (L2CAP_DW_CONGESTED);

	//成功发送，并且此时 Channel 并不拥挤
    return (L2CAP_DW_SUCCESS);
}

//通过 Channel ID 找到 Channel Control Block
tL2C_CCB *l2cu_find_ccb_by_cid (tL2C_LCB *p_lcb, UINT16 local_cid)
{
    tL2C_CCB    *p_ccb = NULL;
#if (L2CAP_UCD_INCLUDED == TRUE)
    UINT8 xx;
#endif

    if (local_cid >= L2CAP_BASE_APPL_CID) //大于或等于 0x0040 说明不是 Fixed Channel
    {
        /* find the associated CCB by "index" */
        local_cid -= L2CAP_BASE_APPL_CID;

        if (local_cid >= MAX_L2CAP_CHANNELS)
            return NULL;

        p_ccb = l2cb.ccb_pool + local_cid; //直接通过地址偏移找到

        /* make sure the CCB is in use */
        if (!p_ccb->in_use)
        {
            p_ccb = NULL;
        }
        /* make sure it's for the same LCB */
        else if (p_lcb && p_lcb != p_ccb->p_lcb)
        {
            p_ccb = NULL;
        }
    }
#if (L2CAP_UCD_INCLUDED == TRUE) //默认是关闭的，既然从上层来的都是 数据包了，我认为不会用到 Fixed Channel
    else
    {
        /* searching fixed channel */
        p_ccb = l2cb.ccb_pool;
        for ( xx = 0; xx < MAX_L2CAP_CHANNELS; xx++ )
        {
            if ((p_ccb->local_cid == local_cid)
              &&(p_ccb->in_use)
              &&(p_lcb == p_ccb->p_lcb))
                break;
            else
                p_ccb++;
        }
        if ( xx >= MAX_L2CAP_CHANNELS )
            return NULL;
    }
#endif

    return (p_ccb);
}


{% endhighlight %}

下面我们来看看 L2CAP 层的处理
