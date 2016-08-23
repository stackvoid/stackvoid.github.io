---
layout: post
category: Bluetooth
title:  蓝牙基本概念
---

## 目录

1. [Introduction](#Intro)
1. [Baseband](#Bb)
1. [高层通信协议](#HighProtocol)
1. [Profile](#Profile)
1. [安全保护机制](#Security)



## <a id="Intro">Introduction</a>

智能硬件的兴起，注定给蓝牙打了一个翻身胜仗(和WLAN)。

## <a id="Bb">Baseband</a>

**Device Addressing:**

*BD_ADDR*, Bluetooth Device Address，  唯一48位地址， 地址分为三部分，LAP（Lower Address Part, 24bits, 如计算Chanel Access Code），UAP（Upper Address Part, 8bits, 计算BT设备调序列），NAP（Non-significant Adress Parts,16bits, 打酱油）

*AM_ADDR*, Active Member Address, 3 bits, Master的AM_ADDR为000. 显然1个Piconet ( a network that is created using a wireless Bluetooth connection)内包括master和slave最多有8个.

*PM_ADDR*, Parked Member Address, 8 bits, 从Active状态进入到Park状态（slave设备只与Piconet时序同步但不传送数据）的BT Device, Piconet 最多容纳256个Park status 的BT Device

*AR_ADDR*, Access Request Address, Piconet内所有Park状态BT Device都会分配AD_ADDR（Device可能分配相同AR_ADDR）, 当Slave从Park回到Active时以此地址向Master要求可传送的Slave-to-Master时槽（Time Slot是一个固定长度的二进制位串，对应一个特定的时间片段，每个站点只能在时间片内发送数据）。

**Physical channel:**
master 和 slave 以 TDD(Time Division Duplex) 传送。Master只在偶数时槽传送数据，Slave只在奇数时传送数据。Master传送封包可扩展至3-5个。

**Frequency Hopping**

避免遭受干扰。Bt将2.4GHz通讯频段切割多个频道，FP技术是将物理通道内的每个时槽上所传送的资料，不断从一个频道跳到另一个频道。
跳频序列（Hopping Sequence）由BD_ADDR决定，同一个Piconet内所有Bt设备，有相同的HS。

**master slave 的时间同步**

所有的BT设备都有一个内部的时序CLKN（Native Clock）。
Piconet内中建立时序同步的方法是以master的内部时序CLK（Master clock）为基准，此Piconet内每个slave都加上时序偏移（offset），以使每个slave的时序与Master的时序互相同步。每次master传送封包到slave上时，都会加上时序偏移（offset）来校正时序。

CLKE（Estimated Clock）为系统内的另一个时序，当Master在呼叫（paging）时，以估计时序CLKE预测被呼叫slave内部时序CLKN，配合已知被呼叫slave的BD_ADDR，尝试与此slave建立同步。

**Scatternet**
Piconet之间的设备通讯。

**数据传输**
bt技术科同时传输语音和数据，原因在于bt支持电路交换（Circuit-switch, SCO连接）和封包交换（Packet-switch, ACL连接）两种资料传输方式。

*SCO连接*, Synchronous Connection-Oriented, Master和Slave一旦建立SCO, 系统会预留固定间隔的时槽给Master与Slave, 其他Slave不能用此SCO上的时槽传送数据, SCO属于简单的点对点（ptp）连接.

*ACL连接*, Asynchronous Connection-Less, 封包交换将上层数据切割成很多封包(packet), ACL利用利用任何非SCO时槽传送数据，系统需要SCO连接，ACL自动让出时槽. Master可同时与多个Slave建立ACL连接，属于点对多(PtM)非对称连接.
*连接数目*, Master和各个Slave间，最多有一条ACL，但可有多条SCO. 在建立SCO连接时必须先建立ACL来传送控制信号.

**语音编码(Voice Coding)**

Bt内的语音编码可以用 PCM(Pulse Code Modulation) 或 CVSD(Continuous Variable Slope Delta Modulation)方式.

**Error Correction**

Bt 的Baseband内共定义了三种Error Correction方法. 1/3FEC(Forward Error Correction)、 2/3FEC、以及 ARQ(Auto Repeat Request)

**bt 封包结构**

Bt封包分为 Access Code, Header, Payload三部分，但并非所有的封包都包含着三部分.

*Access Code*, Preamble(4bits, DC直流补偿, 1010 or 0101, 取决于LSB), Sync World(64位, from BD_ADDR的24位LAP计算来的), Trailer(延长Preamble的DC直流补偿), 长度为68或72位（取决于有没有Trailer）.

- *CAC*, Channel Access Code, Master组建Piconet时, 会分配给Piconet一个唯一的Access Code称为CAC, 所有Piconet内的Active状态的Bt设备使用相同的CAC. Slave 接收封包时，该封包含有CAC与该Slave的AM_ADDR地址，其他Slave收到不含自己AM_ADDR封包时自动弃包.


- *DAC*, Device Access Code, Master发出呼叫paging或Slave回应Master的呼叫Paging时，呼叫信号用的是DAC. DAC由Slave内BD_ADDR的LAP计算来.

- *IAC*, Inquiry Access Code, Master想要查询周围是否有其他Slave时, 会发IAC, IAC分为两种, GIAC(General IAC, 针对一般所有的设备), DIAC(Dedicated IAC, 特定的bt设备).

*Header*, 包含重要的控制信息, 18bits.

 - AM_ADDR, 3bits, 就是Bt地址的AM_ADDR, Master将地址分配给Piconet中不同的Slave, 由于值?000用于Master-to-Slave的广播FHS封包, 让Slave从Active状态进入到Park状态，将丢掉此位.
 - TYPE, 4bits, 封包种类, 如SCO连接或ACL连接等.
 - ARQN, 1bit, 仅对ACL进行流量控制. 拥堵时传回FLOW=0来暂停数据传输. 接收器清空时返回FLOW=1告知设备重新发送传送数据.
 - SEQN, Sequence Number, 1bit, 判断封包是否为重复封包，必须与ARQN互相配合. 用于数据传输流量控制.
 - HEC, Header Error Check, 1bit, 提供用来检查Header在传送过程中是否发生错误.
 
*Payload*, ACL连接内的封包Payload内只有数据, SCO内封包Payload内只有语音, DV封包上的Payload同时有语音和数据.


**Connection State**

Slave共有Active, Sniff, Hold, Park四种连接状态.

- *Active*, Slave和Master互传资料时一般的工作模式. 在Active状态下Slave具有AM_ADDR地址以及与Piconet相同的跳频序列.
- *Sniff*, Slave 省电进入此模式, slave将延长在跳频序列上接收Master信号间隔(间隔由master内的LMP控制). 此时仍保持有AM_ADDR和与Piconet相同的跳频序列.
- *Hold*, 仅支持SCO. Slave 进入Hold模式是为了空出物理通道内的时槽来进行呼叫(Paging), 呼叫扫描(Page Scan), 查询(Inquiry)或加入其它Piconet.
- *Park*, 当Slave不需要传送资料时, 希望更节省电源而又不离开Piconet, 可进入Park模式. 用Master-to-Slave的广播频道BC(Beacon Channel)周期性联系.

功耗大小依次为 Active, Sniff, Hold, Park, Slave. 功耗越低, 动作周期越长, 是以延长Slave的反应时间来节省耗电量.

**中间状态(Intermediate Substate)**

Bt设备建立连接时, 必须经过Intermediate Substate. 中间状态主要分为查询(Inquiry, 不知道周围是否有Bt设备)和呼叫(Paging, 知道周围有Bt设备)两大类.

不管Bt设备在Connection还是Standby(不与设备连接, 处理很简单)工作状态时都能进入Paging或Inquiry状态以便加入某个Piconet. 在Connection状态的设备在数据处理上比较复杂, 当支持ACL连接的设备要进入Inquiry状态时, 必须暂停ACL连接, 将设备转换到Hold或是Park模式才能进行, 当支持SCO连接的设备要进入Inquiry时, Inquiry的信号不可以打断SCO连接上的封包传送, 必须用SCO连接上封包间所空置的时槽.

*连接过程*


|Master       | Slave   
|---------    |---------
|1.Inquiry -> (通过Inquiring(ID packet)) |2.Inquiry Scan -> Inquiry Response ->(FHS Packet) -> 3
|3.Page -> (通过Paging(ID Packet))       |4.Page Scan -> Page Response -> (ID Packet) -> 5
|5.Master Response -> (FHS packet)| 6. Connection -> (ID Packet) -> 7
|7.Connection|

Master发出的查询洗好将现在TrainA上呼叫Slave, 若未收到Slave回应, 会尝试在TrainB上呼叫Slave.

## <a id="HighProtocol">高层通信协议</a>

### LMP
Baseband内的各项功能皆由Link Manager所控制. Link Manager以LMP(Link Manager Protocol)协议与其他设备的Link Manager互相通信. LMP接收来自更高层协议命令, 向下传送到Baseband层.

LMP主要工作时建立与管理LMP连接, 具有以下功能：

- Piconet Management：管理Piconet, 如连接 中断 SCO与ACL连接建立管理 Master和Slave角色转换 处理低功率的Hold与Sniff连接状态等
- Link configuration: 设置系统运行时参数
- Security functions：传输验证与编码的信号 管理连接秘钥.

### HCI

Bt标准定义了HCI(Host Controller Interface), HCI主要是定义主机控制Bt模组的各个指令意义.
HCI的功能分为三个部分, 第一部分的Transport Firmware位于Bt模组内, 控制Bt模组内的Host Controller, 第二部分为Host Driver 位于Host内, 第三部分为实体的Transport Bus, 可为USB, UART, 或是RS232.

### L2CAP

Logical Link Control and Adaptation Ptrotocol, 可接收来自Baseband的事件, 并把这些事件发往上层的SDP, RFCOMM, TCS协议, 同时也能将上层的协议应用切割成较小的封包由Baseband发送.

![L2CAP01][1]

**逻辑通道的建立**
 
 *ACL连接内建立多个逻辑通道*
 
 L2CAP封包与LMP封包传输在不同的ACL连接上. LMP封包发送建立在由Baseband所建立起来的ACL连接上, ACL连接一点建立一直会存在于Master与多个Slave之间. 传输L2CAP封包前, 两个设备必须以ACL连接为基础, 建立并开启逻辑通道, 传送结束后必须关闭逻辑通道.
 
 *每个逻辑通道端点分配CID通道识别码*
 
 ACL连接最多建立65535个逻辑通道, 每个逻辑通道的每个端点都可以用一个通道识别码CID(Channel Identifier)来表示.
 CID, 16bits, 通常是在逻辑通道建立时分配给各个端点.
 
 *逻辑通道的种类*
 
 逻辑通道连接分为
 - 连接导向通道(Connection-Oriented Channel), 每个通道都是双向传输, 且都要求QoS服务品质.
 - 非连接导向通道(Connectionless Channel), 只限单一方向, 用于传输广播数据, 单一CID能够成为多个逻辑通道的端点.
 - 信号通道(Signaling Channel)

![L2CAP02][2]

**L2CAP的功能**

L2CAP是介于高层和底层间的适配层(Adaption), 主要负责两个Bt设备的间传输数据资料时的分组和重组(Segment and Reassembly), 协定多工(Protocol Multiplexing)和协商通道参数等.

*协商通道参数(Negotiation)*

当连接导向建立起来后传输数据前, 必须经过通道参数设定阶段, 这个阶段是设备双方互相协商都同意接受的额通道参数, 通道参数主要为最大区段长度MTU(Maximum Transmission Unit), Flush Timeout以及服务品质QoS(Quality of Service)

*分段和重组(Segment and Reassembly)*

Negotiation后, L2CAP必须将数据资料分割成固定大小的封包(L2CAP区段最长为64Kb)传送给Baseband. Baseband只专注处理来自L2CAP的封包.
L2CAP的区段在传送过程中, 必须等同一区段的所有封包传送结束后, 才能继续传送其他区段的封包. 

*协议多工(Protocol Multiplexing)*

将高层协议的不同种类, 大笑道额封包经过分段后同一封装在L2CAP封包中传给Baseband处理.

### RFCOMM

**主要功能**

RFCOMM是位于L2CAP之上的传输层, RFCOMM上面包括TCP/IP WAP OBEX等协议, 凡是使用Serial Port RS232上面的应用都能操作在RFCOMM协议之上.为两个Bt设备简单P2P通信.

*建立RFCOMM Session*

RFCOMM封包传输前必须建立RCFOMM Session(建立在L2CAP之上). 当发射端希望与接收端通信时, 将先依照信号连接建立起一条逻辑通道(两端各分配一个CID识别码), 设备在每个逻辑通道内建立很多Data Link, 每个Data Link分配一个特定的DLCI(Data Link Connection Identifier)识别码, 建立起RFCOMM Session. 一个Session可同时建立60个Data Link.

### SDP

Service Discovery Protocol, 是应用程序发现网络中可用的服务以及这些服务特性的一种控制机制

### TCS Binary

电话控制协议TCP(Telephone Control Protocol)包括TCS Binary与电话控制AT-Commands(控制移动电话和数据机的组合)两部分, TCS Binary定义了在Bt设备间建立语音盒数据呼叫所需要的呼叫控制指令(Call Control Signaling).

### OBEX

在应用上, 必须将OBEX映射到协曾的RFCOMM及TCP/IP, 成为OBEX over RFCOMM与OBEX over TCP/IP.
采用OBEX协议的典型应用包括Object Push(如交换电子名片) 资料传输(File Transfer)以及多个设备间的资料同步(Synchronization).


## <a id="Profile">Profile</a>

目的是**确保Bluetooth设备的互通性(Interoperability)**.

### 每个Profile实现互通性的方法

从通讯协议层和应用层两层着手, 制定设备操作该Profile时应该遵守的一些规范.

*通讯协议层*

各个Profile定义了不同场景下所涵盖的通讯技术协议架构. 如蓝牙耳机与手机通信用Headset Profile, 笔记本脸上bt手机, 用手机上网, 用的时Dial-up Networking Profile.

### Bluetooth标准定义的各种Profile

主要分为四大类.

- Generic Access Profile与Service Discover Profile. 所有的Bt设备都必须符合GAP的规范.
- 以电话电话关联的Profile, 包括Cordless Telephony Profile 与 Intercom Profile
- 以Serial Port Profile为基础, 发展出相关的Dial-up Networking Profile 与LAN相关的LAN Access profile.
- 以Generic Object Exchange Profile为基础发展使用Object Push的Object Push Profile, 文件传送File Transfer Profile, 及资料同步的Synchronization Profile.

**Generic Access Profile**

是所有Bt设备的基础, 所有bt设备都必须符合GAP的规定, 以确保与其他Bt设备的互通性. GAP描述了Bt设备基本运行方式和参数定义, 主要包括了如何与其他设备连接的Discovery of Bluetooth Devices, Link Management, 安全性等级, 用户界面, 操作模式等.

**Service Discovery Application Profile**

Bt设备搜寻附近有哪些Bt设备或询问某特定的Bt设备提供哪些服务.

**TCS-Based**

*Cordless Telephony Profile*

移动电话与基座的通讯.

*Intercom Profile*

手机之间用蓝牙通信.

**Serial Port Profile**

*Headset Profile*

蓝牙耳机.减少电线的干扰.

*Dial-up Networking Profile*

电脑通过手机蓝牙用手机的网络上网.

*LAN Access Profile*

路由终端, 用蓝牙连接各种设备, 使设备能联网. （无线路由器功能）

**Generic Object Exchange Profile**

以Serial Port Profile为基础, 并在GOEP发展处的Object Push Profile, File Transfer Profile(文件或文件夹的传输, 浏览控制另一个设备的文件系统), Synchronization Profile等.

## <a id="Security">安全保护机制</a>

在Bt标准内, 发出验证要求的设备称为Claimant, 负责验证的设备称为Verifier.

### 安全机制的模式

GAP将安全保护机制分为三种模式

- Security Mode 1：Non-secure
- Security Mode 2：Service level enforced Security
- Security Mode 3：Link level enforced Security(PIN号码)













  [1]: https://lh5.googleusercontent.com/-CdenHSvFT0Q/U1d07ACNtPI/AAAAAAAAAGc/MJyg_BMVuPI/s0/1.jpg "1.jpg"
  [2]: https://lh5.googleusercontent.com/-hABu5utNcLE/U1d1DbYU0CI/AAAAAAAAAGk/xUp4nPwerc0/s0/2.jpg "2.jpg"
