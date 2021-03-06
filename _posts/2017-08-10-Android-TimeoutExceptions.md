---
layout: post
title:  " Android TimeoutExceptions及 wakelock"
date:   2017-08-10 20:00:00 +0800
categories: Android 
tags: Android 
author: chenjy
---



* content
{:toc}


本篇简单介绍调试机器人出现的`TimeoutExceptions`问题 `Android 4.4`。




## 问题描述

`机器人介绍`：讲解机器人软件部分 分为双层结构。机器人胸口屏幕负责 UI 交互以及和服务器的通信。机器人底层`Android v4.4` 主板负责机器人运动控制。

两层软件 通过网线进行通讯。在机器人的讲解过程中经常底层`Android` 主板上的App经常会出现`TimeoutExceptions` 问题崩溃。

> java.util.concurrent.TimeoutException: com.android.internal.os.BinderInternal$GcWatcher.finalize() timed out after 10 seconds

## 解决方案

`GcWatcher.finalize`, `BinderProxy.finalize` 和 `PlainSocketImpl.finalize` 中的一类`TimeoutExceptions`。这个异常90%都是发生在`4.3`、`4.4`的`android`系统上。

这个问题的根源在于设备会`Goes to Sleep`一会儿，就是说操作系统会通过熄屏、降低cpu循环等方式降低电量消耗，进入休眠状态。它是通过在内核层暂停进程的方式来实现的。这可能发生在常规`app`运行的过程中, 但是会停在一次内核调用上，比如内核层的上下文切换。这就是`Dalvik GC`参与最初所说`TimeoutExceptions`问题的方式。

`Dalvik GC`的基本工作方式就是，在`GC`循环中将收集到的一系列的对象去销毁。

如果有一个后台运行的进程，在运行过程中，对象被创建、使用以及被收集(以释放内存)。一般的，应用不会使用`Wakelock`，因为会很耗电并且也没必要，这意味着应用会不时的执行GC动作。通常情况下，GC动作会正常的执行完成而不会被挂起。但是，有些时候(很稀少)，操作系统会在GC动作的过程中进入休眠。如果你的应用运行时间足够长，它就有可能发生。现在，想一下GC循环中的有关时间戳的逻辑：有可能发生，设备开始进行GC，并且在处理系统对象销毁(native层的destroy())的过程中进入休眠， 然后被唤醒，恢复运行，记录现在的时间戳，也就是说这次GC动作花费的时间=销毁动作执行时长+休眠时长。如果休眠时间超过10s, 就会抛出`concurrent.timeout`异常。

这个问题不能完全避免，只要你的应用在后台运行。我们可以通过调用`wakelock`减少设备休眠。

## `wakelock`

对于系统来说，系统为了节省电量，`cpu`在没有任务很忙的时候会进入休眠状态。当有任务需要cpu高效执行的时候，就会给cpu加一个`wakelock`锁，我们给CPU加一个锁，系统就不会处于休眠状态。


`AndroidManifest.xml`:

```java

<uses-permission android:name="android.permission.WAKE_LOCK"/>
<uses-permission android:name="android.permission.DEVICE_POWER"/>

```

```java


    private boolean iswakeLock = true;// 是否常亮   
    private WakeLock wakeLock;  

	@Override
		protected void onResume() {
			// TODO Auto-generated method stub
			PowerManager pm = (PowerManager) getSystemService(POWER_SERVICE);
			wakeLock = pm.newWakeLock(PowerManager.SCREEN_BRIGHT_WAKE_LOCK
					| PowerManager.ON_AFTER_RELEASE, "MyWakelockTag");

			if (iswakeLock) {
				wakeLock.acquire();
			}
			super.onResume();

		}

    @Override
    protected void onPause() {
        // TODO Auto-generated method stub
        super.onPause();
        if (wakeLock != null) {
            wakeLock.release();
        }
    }


```

在`onRusume`方法中将获得到的锁使用`acquire()`方法来保持唤醒，在`onPause`方法中使用`release()`方法来释放掉该锁。

`锁类型`:

* `PARTIAL_WAKE_LOCK`:保持CPU 运转，屏幕和键盘灯有可能是关闭的

* `SCREEN_DIM_WAKE_LOCK`：保持CPU 运转，允许保持屏幕显示但有可能是灰的，允许关闭键盘灯

* `SCREEN_BRIGHT_WAKE_LOCK`：保持CPU 运转，允许保持屏幕高亮显示，允许关闭键盘灯

* `FULL_WAKE_LOCK`：保持CPU 运转，保持屏幕高亮显示，键盘灯也保持亮度

* `ACQUIRE_CAUSES_WAKEUP`：正常唤醒锁实际上并不打开照明。相反，一旦打开他们会一直仍然保持。当获得wakelock，这个标志会使屏幕或/和键盘立即打开。一个典型的使用就是可以立即看到那些对用户重要的通知

* `ON_AFTER_RELEASE`：设置了这个标志，当`wakelock`释放时用户`activity`计时器会被重置，导致照明持续一段时间。如果你在`wacklock`条件中循环，这个可以用来减少闪烁


## `update` `2018/04/06`

###  keep the screen on

控制屏幕一直处于常亮状态

1. 在`Activity`中使用  `FLAG_KEEP_SCREEN_ON`

```java

public class MainActivity extends Activity {
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    getWindow().addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON);
  }

```

2. 或是在`xml` 使用参数`keepScreenOn`

```xml

<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:keepScreenOn="true">
    ...
</RelativeLayout>

```

### keep the CPU on

控制cpu一直处于工作状态

` manifest file`:

```xml

<uses-permission android:name="android.permission.WAKE_LOCK" />

```

```java

PowerManager powerManager = (PowerManager) getSystemService(POWER_SERVICE);
WakeLock wakeLock = powerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK,
        "MyWakelockTag");
wakeLock.acquire();

//  wakelock.release() /* release wakelock*/

```

### Using WakefulBroadcastReceiver

`WakefulBroadcastReceiver` 是一种特殊的`broadcast receiver`为你的`app`提供`WAKE_LOCK`管理服务，确保在耗时任务完成之前设备不会熄屏。

* 通过方法 `startWakefulService(Context, Intent)`来启动服务，启动后`WakefulBroadcastReceiver` 将自动创建一个`wakelock` ，并在`Intent`附上一个`wakelock Id`来区分 。

* 最后必须调用 `completeWakefulIntent(intent)` 释放`wakelock`。

`WakefulBroadcastReceiver`,`Demo`

`AndroidManifest.xml`

```xml

<receiver android:name=".SimpleWakefulReceiver"></receiver>

```

启动 `WakefulBroadcastReceiver`

```java

public class MyWakefulReceiver extends WakefulBroadcastReceiver {            
    @Override    
    public void onReceive(Context context, Intent intent) {
        Intent service = new Intent(context, MyWakefulReceiver.class);      
        startWakefulService(context, service);    
    }
}

```

在耗时操作结束之后释放`wakelock`

```java

public class MyWakefulService extends IntentService {    
    public MyWakefulService() {        
        super("MyWakefulService");    
    } 

    @Override    
    protected void onHandleIntent(Intent intent) {        
        ...
        // 执行耗时任务
    
        // 任务结束释放唤醒锁
        MyWakefulReceiver.completeWakefulIntent(intent);    
    }
}

```



