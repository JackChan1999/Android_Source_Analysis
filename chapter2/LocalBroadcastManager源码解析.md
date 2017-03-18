#LocalBroadcastManager源码解析

##1.简介

LocalBroadcastManager是Android v4兼容包提供的应用内广播发送与接收的工具类。BroadcastReceiver的通信是基于Binder机制，而LocalBroadcastManager的核心是基于Handler机制。

相比BroadcastReceiver的广播，LocalBroadcastManager有以下几点优点。

- 广播数据只在本应用内传播，不用担心数据泄露，
- 广播数据不用担心别的应用伪造广播，更加安全。
- 因为只在应用内广播，所以更加的高效。

##2.基本使用方法

###2.1 自定义 BroadcastReceiver 子类

```java
public class LocalBroadcastReceiver extends BroadcastReceiver {

    @Override
    public void onReceive(Context context, Intent intent) {
        //处理广播信息
    }
}
```

###2.2 注册广播

```java
LocalBroadcastReceiver localReceiver = new LocalBroadcastReceiver();  
IntentFilter filter = new IntentFilter(ACTION_LOCAL_SEND);
LocalBroadcastManager.getInstance(context).registerReceiver(localReceiver, filter);  
```

###2.3 发送广播

```java
LocalBroadcastManager.getInstance(context).sendBroadcast(new Intent(ACTION_LOCAL_SEND));  
```


###2.4 取消广播注册

```java
LocalBroadcastManager.getInstance(context).unregisterReceiver(localReceiver); 
```

##3.源码解析

###3.1 LocalBroadcastManager原理概要


LocalBroadcastManager使用单例模式对象，初始化时会在内部初始化一个Handler对象用来接受广播。注册广播时，会将自定义的BroadcastReceiver对象和IntentFilter对象保存到HashMap中。发送广播时，则根据IntentFilter的Action值从已保存的HashMap找到对应接受者，并发送Handler消息去执行receiver的onReceive方法。

LocalBroadcastManager核心代码为以下四个函数。

- registerReceiver(BroadcastReceiver receiver, IntentFilter filter)    //注册广播函数
- unregisterReceiver(BroadcastReceiver receiver)    //取消注册函数
 - sendBroadcast(Intent intent)		//发送广播
  - executePendingBroadcasts()   //处理接受到的广播



###3.2 LocalBroadcastManager基本数据结构

LocalBroadcastManager需要保存三样东西，一个是 **mReceivers**, 用来保存已注册的自定义的receiver和intentFilter。一个是 **mActions** 键值对，保存action和ReceiverRecord列表的键值对。一个是 **mPendingBroadcasts** , 用来保存待通知的receiver对象。

```java
 //注册广播Record类
 private static class ReceiverRecord {
        final IntentFilter filter;
        final BroadcastReceiver receiver;
        boolean broadcasting;
        
        ReceiverRecord(IntentFilter _filter, BroadcastReceiver _receiver) {
            filter = _filter;
            receiver = _receiver;
        }
        ...
 }

private final HashMap<BroadcastReceiver, ArrayList<IntentFilter>> mReceivers
        = new HashMap<BroadcastReceiver, ArrayList<IntentFilter>>();
        
private final HashMap<String, ArrayList<ReceiverRecord>> mActions
        = new HashMap<String, ArrayList<ReceiverRecord>>();

private final ArrayList<BroadcastRecord> mPendingBroadcasts
        = new ArrayList<BroadcastRecord>();

//待广播的Record类
private static class BroadcastRecord {
    final Intent intent;
    final ArrayList<ReceiverRecord> receivers;

    BroadcastRecord(Intent _intent, ArrayList<ReceiverRecord> _receivers) {
        intent = _intent;
        receivers = _receivers;
    }
}
    
```




###3.3  注册广播

将需要注册的receiver对象和该receiver需要监听的filter保存到 **mReceivers** 和 **mPendingBroadcasts** 中。

```java
/**
 * Register a receive for any local broadcasts that match the given IntentFilter.
 *
 * @param receiver The BroadcastReceiver to handle the broadcast.
 * @param filter Selects the Intent broadcasts to be received.
 *
 * @see #unregisterReceiver
 */
public void registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
    synchronized (mReceivers) {
        ReceiverRecord entry = new ReceiverRecord(filter, receiver);
        ArrayList<IntentFilter> filters = mReceivers.get(receiver);
        if (filters == null) {
            filters = new ArrayList<IntentFilter>(1);
            mReceivers.put(receiver, filters);    //保存receiver和filter到List
        }
        filters.add(filter);
        for (int i=0; i<filter.countActions(); i++) {
            String action = filter.getAction(i);
            ArrayList<ReceiverRecord> entries = mActions.get(action);
            if (entries == null) {
                entries = new ArrayList<ReceiverRecord>(1);
                mActions.put(action, entries);   //保存到action和ReceiverRecord到HashMap
            }
            entries.add(entry);
        }
    }
}
```

###3.4  取消广播注册

根据receiver对象移除 **mReceivers** 和 **mPendingBroadcasts** 中对应的对象。

```java
/**
 * Unregister a previously registered BroadcastReceiver.  All
 * filters that have been registered for this BroadcastReceiver will be
 * removed.
 *
 * @param receiver The BroadcastReceiver to unregister.
 *
 * @see #registerReceiver
 */
public void unregisterReceiver(BroadcastReceiver receiver) {
    synchronized (mReceivers) {
        //从mReceivers中移除
        ArrayList<IntentFilter> filters = mReceivers.remove(receiver);
        if (filters == null) {
            return;
        }
        for (int i=0; i<filters.size(); i++) {
            IntentFilter filter = filters.get(i);
            for (int j=0; j<filter.countActions(); j++) {
                String action = filter.getAction(j);
                ArrayList<ReceiverRecord> receivers = mActions.get(action);
                if (receivers != null) {
                    for (int k=0; k<receivers.size(); k++) {
                        if (receivers.get(k).receiver == receiver) {
                            receivers.remove(k);
                            k--;
                        }
                    }
                    if (receivers.size() <= 0) {
                        mActions.remove(action);  //从mActions中移除
                    }
                }
            }
        }
    }
}
```

###3.5  通过Handler发送广播

发送广播时，先根据intent中的action到**mActions**中找到对应的记录，然后再完整匹配filter里面的各个字段，若匹配成功，则将对应的receiver添加的**mPendingBroadcasts**列表中，等待handler对象的handleMessage()方法处理。

```java
/**
 * Broadcast the given intent to all interested BroadcastReceivers.  This
 * call is asynchronous; it returns immediately, and you will continue
 * executing while the receivers are run.
 *
 * @param intent The Intent to broadcast; all receivers matching this
 *     Intent will receive the broadcast.
 *
 * @see #registerReceiver
 */
public boolean sendBroadcast(Intent intent) {
    synchronized (mReceivers) {
        final String action = intent.getAction();
        final String type = intent.resolveTypeIfNeeded(
                mAppContext.getContentResolver());
        final Uri data = intent.getData();
        final String scheme = intent.getScheme();
        final Set<String> categories = intent.getCategories();

		  ...

        //根据intent的action寻找ReceverRecord
        ArrayList<ReceiverRecord> entries = mActions.get(intent.getAction());
        
        if (entries != null) {
 
            ArrayList<ReceiverRecord> receivers = null;
            for (int i=0; i<entries.size(); i++) {
            
                ReceiverRecord receiver = entries.get(i);
					//相同的receiver,只添加一次 
                if (receiver.broadcasting) {
                    continue;
                }

                int match = receiver.filter.match(action, type, scheme, data,
                        categories, "LocalBroadcastManager");

                if (match >= 0) {
                    if (receivers == null) {
                        receivers = new ArrayList<ReceiverRecord>();
                    }
                    receivers.add(receiver);
                    //标记为已添加，待广播状态 
                    receiver.broadcasting = true;
                } else {
 						...
                }
            }

            if (receivers != null) {
            
            		//receivers添加完成后，将broadcasting状态回归
                for (int i=0; i<receivers.size(); i++) {
                    receivers.get(i).broadcasting = false;
                }
                
                //添加到待广播列表
                mPendingBroadcasts.add(new BroadcastRecord(intent, receivers));
                
                //若无正在处理的消息，则handler发送广播消息
                if (!mHandler.hasMessages(MSG_EXEC_PENDING_BROADCASTS)) {
                    mHandler.sendEmptyMessage(MSG_EXEC_PENDING_BROADCASTS);
                }
                
                return true;
            }
        }
    }
    return false;

```

###3.6  Handler接受和消费广播

在handler对象的handleMessage()方法中遍历 **mPendingBroadcasts** 列表, 依次循环调用其中的onReceive()方法，并将intent中的数据传入，从而消费广播信息。

```java

private LocalBroadcastManager(Context context) {
    mAppContext = context;
    mHandler = new Handler(context.getMainLooper()) {

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_EXEC_PENDING_BROADCASTS:
                    executePendingBroadcasts();
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    };
}

private void executePendingBroadcasts() {
    while (true) {
        BroadcastRecord[] brs = null;
        synchronized (mReceivers) {
            final int N = mPendingBroadcasts.size();
            if (N <= 0) {
                return;
            }
            //拷贝数据到brs数组
            brs = new BroadcastRecord[N];
            mPendingBroadcasts.toArray(brs);
            mPendingBroadcasts.clear();
        }
        
        for (int i=0; i<brs.length; i++) {
            BroadcastRecord br = brs[i];
            for (int j=0; j<br.receivers.size(); j++) {
                //循环数组里的内容，调用其onReceive方法，消费广播内容
                br.receivers.get(j).receiver.onReceive(mAppContext, br.intent);
            }
        }
}
```
##4.总结

LocalBroadcastManager在应用内使用起来比较简单高效，但是其也是有一些缺点的。比如LocalBroadcastManager并不支持静态注册广播，也不支持有序广播的一些功能。不过如果仅仅是普通广播通信也是够用了。
