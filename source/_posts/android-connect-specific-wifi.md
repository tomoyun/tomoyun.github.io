---
title: Android连接指定的WiFi热点
date: 2018-02-19 20:21:15
categories:
- Android开发
tags:
- Android
- WifiManager
---

“Android自动连接指定的WiFi热点”，看上去这是个再基础不过的功能了。很多人都觉得很简单，网上也有大量的资料，但是真拿代码来开发实践，却总是发现差一点。本文就来详细叙述这其中存在什么样的玄机。
<!--more-->

## 常规解决方案
在Android系统中，WifiManager类用来管理WiFi连接的各个方面。翻看WifiManager的API文档，发现了这几个方法:

- <font color=DodgerBlue>int addNetwork(WifiConfiguration config)</font>  
Add a new network description to the set of configured networks.
- <font color=DodgerBlue>int updateNetwork(WifiConfiguration config)</font>  
Update the network description of an existing configured network.
- <font color=DodgerBlue>boolean removeNetwork(int netId)</font>  
Remove the specified network from the list of configured networks.
- <font color=DodgerBlue>boolean enableNetwork(int netId, boolean attemptConnect)</font>  
Allow a previously configured network to be associated with.
- <font color=DodgerBlue>boolean reassociate()</font>  
Reconnect to the currently active access point, even if we are already connected.
- <font color=DodgerBlue>boolean reconnect()</font>  
Reconnect to the currently active access point, if we are currently disconnected.

推测代码大抵就是这些API调用的组合，网上的例子也确实如此: 

```
WifiConfiguration wifiConfig = createWifiConfig(ssid, password,type);
WifiConfiguration tempConfig = isExsits(ssid);
if (tempConfig != null) {
    wifiManager.removeNetwork(tempConfig.networkId);
}
int netID = wifiManager.addNetwork(wifiConfig);
boolean enabled = wifiManager.enableNetwork(netID, true);
boolean connected = wifiManager.reconnect();
```
注: 这里省略了创建WifiConfiguration的代码实现

运行一下试试，手机顺利连接上指定WiFi，收工 ^ ^!

一个小问题: Android6.0及以上系统是不允许删除或者更新其他应用创建的Wifi网络的。如果WifiConfig是由用户或其他应用创建的，直接利用netID进行连接。

## 差一点
当然如果真这么顺利就解决了这个问题的话，那真没必要在这啰里吧嗦了。多拿几台手机来测试测试，发现这段代码真是奇了怪了， 有时能连接上Wifi, 有时却又不能，有时连上的并不是我们指定的Wifi热点。

这是为什么呢？还真没法一下子说明白，怀疑和各个Android厂商定制的ROM有关。Android系统的兼容性真是搞死人...

再翻看WifiManager的API文档，上上下下就这么几个方法，前前后后颠倒顺序都试一遍，效果还是一样，时灵时不灵。到了这里，真的走投无路了，只能挖出WifiManager的系统源码开始啃了。发现上面API方法的实现都只是调用WifiService的对应方法。WifiService和WifiManager通过AIDL和Messenger(本质还是AIDL, 进行了封装)进行通信。接着往下翻，这是啥？

```
/**
 * Connect to a network with the given configuration. The network also
 * gets added to the list of configured networks for the foreground user.
 *
 * For a new network, this function is used instead of a
 * sequence of addNetwork(), enableNetwork(), saveConfiguration() and
 * reconnect()
 *
 * @param config the set of variables that describe the configuration,
 *            contained in a {@link WifiConfiguration} object.
 * @param listener for callbacks on success or failure. Can be null.
 * @throws IllegalStateException if the WifiManager instance needs to be
 * initialized again
 *
 * @hide
 */
@SystemApi
public void connect(WifiConfiguration config, ActionListener listener) {
    if (config == null) throw new IllegalArgumentException("config cannot be null");
    // Use INVALID_NETWORK_ID for arg1 when passing a config object
    // arg1 is used to pass network id when the network already exists
    getChannel().sendMessage(CONNECT_NETWORK, WifiConfiguration.INVALID_NETWORK_ID,
            putListener(listener), config);
}
```

这不正是我们想要的吗？ 直接调用这个方法不就完了么？还要啥addNetwork、enableNetwork、reconnect？ 统统不用调用，简直爽的不要不要的。  
慢着再仔细看看，@hide、@SystemApi、什么鬼？ 就说这是系统隐藏API，不让用。凭啥系统设置就能用，不管三七二十一，先拿来试试，果然和系统的WiFi连接效果一致，迅速连上指定的WiFi热点，比之前的那些个API方法强太多。再看这个connect方法的实现，只是通过AysncChannel发消息给WifiService，具体的操作还是在WifiService中。

```
private synchronized AsyncChannel getChannel() {
    if (mAsyncChannel == null) {
        Messenger messenger = getWifiServiceMessenger();
        if (messenger == null) {
            throw new IllegalStateException(
                    "getWifiServiceMessenger() returned null!  This is invalid.");
        }

        mAsyncChannel = new AsyncChannel();
        mConnected = new CountDownLatch(1);

        Handler handler = new ServiceHandler(mLooper);
        mAsyncChannel.connect(mContext, handler, messenger);
        try {
            mConnected.await();
        } catch (InterruptedException e) {
            Log.e(TAG, "interrupted wait at init");
        }
    }
    return mAsyncChannel;
}
```

再看AsyncChannel的构造过程，其中有一个handler和一个messenger。handler是WifiManager的ServiceHandler对象。messenger通过调用WifiService的getWifiServiceMessenger函数获取，看源码也可以理解为是一个Handler。AsyncChannel通过这两个handle建立WifiManager和WifiService之间的双向通信连接。


## 不得已的解决方案

根据上面的分析，看来通过调用系统的隐藏API方法，是能够帮助我们解决掉一部分兼容性问题的，且效果要好于Android常规的API。然而Android系统版本和厂商繁多，不能全指望这个隐藏方法，况且在低版本的系统中并没有这个方法。
它给我们带来了思路: 在不同的Android版本中，如果可以的话优先通过反射使用系统的某些方法直接连接WiFi热点。 到最后再通过常规的解决方案进行连接。这种融合方案虽然不能百分百保证实现功能，但可以尽可能的提高连接成功率。

下面这段代码是在Android 4.3以上系统通过反射调用上面的系统隐藏方法连接WiFi。

```
public boolean connectJellyBean(int networkId) {
    if(mReflectMethod) {
        for (Method methodSub : mWifiManager.getClass().getDeclaredMethods()) {
            if ("connect".equalsIgnoreCase(methodSub.getName())) {
                Class<?>[] types = methodSub.getParameterTypes();
                if (types != null && types.length > 0) {
                    if ("int".equalsIgnoreCase(types[0].getName())) {
                        mConnectMethod = methodSub;
                        break;
                    }
                }
            }
        }
        mReflectMethod = false;
    }

    if (mConnectMethod != null) {
        try {
            mConnectMethod.invoke(mWifiManager, networkId, null);
        } catch (Exception e) {
            return false;
        }
    }
    return true;
}
```
