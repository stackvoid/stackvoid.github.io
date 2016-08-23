---
layout: post
title: Bluedroid Enable流程分析
categories: [Bluetooth]
tags: [Bluetooth]
---



本文基于Android4.4代码，对Bluetooth BR/EDR Enable Process的分析。Bluetooth Enable Process比较复杂，层次多，本文对照logcat输出的Bluetooth相关log来分析阅读代码。

首先从总体上介绍以下Enable Process。UI层的入口是Settings，触发Bluetooth开关，启动Bluetooth Enable Process，然后由Bluedroid去enable Bluetooth hardware。当Bluetooth hardware enabled，这个enabled消息会一层层从Bluedroid上传到UI层，Settings收到这个消息就可以更新Bluetooth开关的状态了。具体过程如下图：

![Bluetooth Enable](http://stackvoid.qiniudn.com/2014-04-28-BluetoothEnable%20.png)

**Settings/BluetoothEnable类**

UI上的Bluetooth开关。

方法BluetoothEnabler获得LocalBluetoothManager对象，如果该设备支持蓝牙，则打开蓝牙开关。得到代表local device的BluetoothAdapter，再调用BluetoothAdapter::enable()。

**BluetoothAdapter**

BluetoothAdapter类直接调用BluetoothManagerService::enable(), 这个类是个Wrapper。

**BluetoothManagerService**

BluetoothManagerService利用Binder机制去connect AdapterService，最终导致AdapterService::enable()被调用。BluetoothManagerService还会向AdapterService注册callback函数，用于接收Adapter State Change消息。

**AdapterService**

1. AdapterService维护着一个状态机AdapterState，所有工作都是通过驱动状态机来完成的。AdapterState收到AdapterService发过来的USER_TURN_ON消息，就会调用AdapterService::processStart()来启动Profie Services的初始化和Bluetooth hardware enable process。此时Bluetooth Adapter的状态是BluetoothAdapter.STATE_TURNING_ON。

2. 每一个profile都有一个service。每个profile service启动完成后，都会通知AdapterService。当AdapterService::processProfileServiceStateChanged()确认所有的profile services都启动完成了，就会给状态机AdapterState发AdapterState.STARTED消息。

3. 状态机AdapterState::PendingCommandState::processMessage()收到AdapterState.STARTED消息后就立刻调用AdapterService::enableNative()。

4. AdapterService::enableNative()就是用来enable Bluetooth的Bluetooth JNI接口。com_android_bluetooth_btservice_AdapterService.cpp里的enableNative()会调用Bluetooth HAL的bt_interface_t->enable()。

5. Bluedroid用btif_enable_bluetooth()来实现了Bluetooth HAL的enable(),
如描述btif_enable_bluetooth()的作用就是Performs chip power on and kickstarts OS scheduler，主要调用bte_main_enable()函数（调用GKI_create_task()）来完成。

6. GKI：统一内核接口 |
BTE栈：|
BTU栈：BTU栈开始前必须调用BTE栈初始化。在代码/external/bluetooth/bluedroid/hci/:HCI library实现。其中/external/bluetooth/bluedroid/hci/src/bt_hw.c中加载了libbt-vendor.so库，它由/device/common/libbt里面的对应vendor生成，初始化了最重要的bt_vnd_if。 通过bt_vnd_if->init将bluedroid的回调函数传过去。而/external/bluetooth/bluedroid/hci/src/bt_hci_bdroid.c中的bt_hc_interface_t包装了bt_vnd_if，提供给BTE调用。/external/bluetooth/bluedroid/hci/src/bt_hw.c中定义了一些vendor调用的函数。 /external/bluetooth/bluedroid/main/bte_main.c中是BTE核心栈的初始化和关闭代码。其中的bt_hc_if就是上面说的bt_hc_interface_t.其中的bte_main_hci_send是由上层栈调用发送msg的。其中的/external/bluetooth/bluedroid/btif/src/bluetooth.c是硬件抽象层HAL的实现.而/external/bluetooth/bluedroid/btif/src/btif_core.c是连接HAL与BTE的核心函数实现，在bluetooth.c中调用了其中的很多函数。即bluetooth.c调用btif_core.c封装的BTA操作。

7. 当Bluedroid真正完成了enable Bluetooth hardware，就通过btif_enable_bluetooth_evt()中的HAL_CBACK调用Bluetooth JNI的adapter_state_change_callback()，这样就把BT_STATE_ON消息传递给了状态机AdapterState。

8. AdapterState会把Bluetooth Adapter的状态转换到BluetoothAdapter.STATE_ON，并通过AdapterState::notifyAdapterStateChanged()通知AdapterService。

9. AdapterService::updateAdapterState()会通过callback函数通知BluetoothManagerService，Adapter状态改变了。

10. BluetoothManagerService确认状态发生了改变就会发出一个BluetoothAdapter.ACTION_STATE_CHANGE的intent。


11. Settings的BluetoothEnabler收到这个intent之后，就会去更新UI上Bluetooth开关的状态。

到此为止，Bluetooth的Enable状态就分析完了。
