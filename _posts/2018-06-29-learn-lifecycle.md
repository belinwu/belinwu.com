---
layout: post
title: 初学Android架构组件之Lifecycle
date: 2018-06-29 13:54
tags: Lifecycle Jetpack
categories: Android
---

在开发应用时，我们可能会基于一系列的生命周期实现某种功能。为了复用，也为了不让应用组件变得很臃肿，实现该功能时会选择与生命周期组件解藕，独立成一种组件。这样能够很方便地在应用组件中使用，比如：`Activity`、`Fragment` 或 `Service`。

Android 官方把它叫做 lifecycle-aware 组件，这类组件能够感知应用组件生命周期的变化，避免一连串显性的生命周期方法委托调用。虽然可以创建基类，在基类中进行委托调用，但由于 Java 是单继承的，就会存在一定的限制。不然发现，这是一次组合优于继承的实践。

为了方便开发者创建 lifecycle-aware 组件，`androidx.lifecycle` 包提供了一些类与接口。

## Lifecycle

`Lifecycle` 类表示Android应用组件的生命周期，是被观察者。这是一个抽象类，它的实现是 `LifecycleRegistry` 类。另外，它通过使用两类数据来跟踪应用组件的生命周期变化，一种是事件，另一种是状态。

`Lifecycle.Event` 表示生命周期的事件，与应用组件的生命周期回调一一对应，这些事件分别是：`ON_CREATE`、`ON_START`、`ON_RESUME`、`ON_PAUSE`、`ON_STOP`、`ON_DESTROY`、`ON_ANY`。最后一种事件可以代表前面任意一种。

举个例子，当 `Activity` 的 `onCreate()` 生命周期方法被调用时会产生 `ON_CREATE` 事件，观察者可以监听该事件以便处理`Activity`此时的生命周期。

`Lifecycle.State` 表示生命周期的状态，一共有5种，分别是：`INITIALIZED`、 `DESTROYED`、`CREATED`、`STARTED`、`RESUMED`。

应用组件初始化之后进入 `INITIALIZED` 状态，在 `onCreate()` 生命周期方法调用后进入 `CREATED` 状态，在 `onStart()` 生命周期方法调用后进入 `STARTED` 状态，在 `onResume()` 生命周期方法调用后进入 `RESUMED` 状态。

事件与状态之间的具体变化关系如下图所示：

![生命周期事件与状态变化关系图（来自官网）](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e5365273efb0~tplv-t2oaga2asx-image.image)

`Lifecycle` 对象有3个方法：

- 添加观察者：`void addObserver(LifecycleObserver observer)`。
- 删除观察者：`void removeObserver(LifecycleObserver observer)`。
- 获取当前状态：`State getCurrentState()`。

## LifecycleOwner

`LifecycleOwner` 接口表示生命周期所有者，即拥有生命周期的应用组件。通过调用 `getLifecycle()` 方法能够获得它所拥有的`Lifecycle` 对象。

`FragmentActivity` 和 `Fragment` 均已实现该接口，可以直接使用。当然开发者也可以自定义。`LifecycleService` 和 `ProcessLifecycleOwner` 是另外两个内置的实现类。

## LifecycleObserver

标记接口 `LifecycleObserver` 表示生命周期观察者，是 lifecycle-aware 组件。

## 小试牛刀

### 添加依赖

新建一个 Android Studio 项目，在 `build.gradle` 文件里添加 `google()` 仓库。

```groovy
allprojects {
    repositories {
        google()
        jcenter() 
    }
}
```

在 app 模块的 `build.gradle` 文件里添加依赖。

```groovy
dependencies {
    def version = "2.0.0-alpha1"
    implementation "androidx.lifecycle:lifecycle-runtime:$version"
    annotationProcessor "androidx.lifecycle:lifecycle-compiler:$version"
}
```

### 例子1：打印生命周期方法被调用日志

实现观察者。

```java
class MyObserver implements LifecycleObserver {

  private static final String TAG = MyObserver.class.getSimpleName();
  
  @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
  public void onCreate() {
      Log.d(TAG, "onCreate called");
  }
  
  @OnLifecycleEvent(Lifecycle.Event.ON_START)
  public void onStart() {
      Log.d(TAG, "onStart called");
  }

  @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
  public void onResume() {
      Log.d(TAG, "onResume called");
  }

  @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
  public void onPause() {
      Log.d(TAG, "onPause called");
  }

  @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
  public void onStop() {
      Log.d(TAG, "onStop called");
  }

  @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
  public void onDestroy() {
      Log.d(TAG, "onDestroy called");
  }

  @OnLifecycleEvent(Lifecycle.Event.ON_ANY)
  public void onAny() {
      Log.d(TAG, "onCreate | onStart | onResume | onPause | onStop | onDestroy called");
  }
}
```

在 `Activity` 中使用该观察者。

```java
public class MyActivity extends AppCompatActivity {

  @Override
  protected void onCreate(Bundle savedInstanceState) {
     // ...
     getLifecycle().addObserver(new MyObserver());
  }
}
```

### 例子2：网络变化观察者

实现这样一个 lifecycle-aware 组件，它能够：在 `onCreate()` 生命周期中动态注册网络变化 `BroadcastReceiver`，在 `onDestory()` 生命周期中注销广播接收者，在收到网络变化时能够判断出网络是如何变化的，并通过回调告知使用者。

实现观察者。

```kotlin
package com.samelody.samples.lifecycle

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Context.CONNECTIVITY_SERVICE
import android.content.Intent
import android.content.IntentFilter
import android.net.ConnectivityManager
import android.net.ConnectivityManager.CONNECTIVITY_ACTION
import android.net.ConnectivityManager.EXTRA_NETWORK_TYPE
import androidx.lifecycle.Lifecycle.Event.ON_START
import androidx.lifecycle.Lifecycle.Event.ON_STOP
import androidx.lifecycle.LifecycleObserver
import androidx.lifecycle.OnLifecycleEvent

/**
 * The network observer.
 *
 * @author Belin Wu
 */
class NetworkObserver(private val context: Context) : LifecycleObserver {

    /**
     * The network receiver.
     */
    private val receiver = NetworkReceiver()

    /**
     * The last type of network.
     */
    private var lastType = TYPE_NONE

    /**
     * The network type changed listener.
     */
    var listener: OnNetworkChangedListener? = null

    @OnLifecycleEvent(ON_START)
    fun onStart() {
        val filter = IntentFilter()
        filter.addAction(CONNECTIVITY_ACTION)
        context.registerReceiver(receiver, filter)
    }

    @OnLifecycleEvent(ON_STOP)
    fun onStop() {
        context.unregisterReceiver(receiver)
    }

    companion object {

        /**
         * The network type: None.
         */
        const val TYPE_NONE = -1

        /**
         * The network type: Mobile.
         */
        const val TYPE_MOBILE = 0

        /**
         * The network type: Wi-Fi.
         */
        const val TYPE_WIFI = 1
    }

    /**
     * The network receiver.
     */
    inner class NetworkReceiver : BroadcastReceiver() {

        override fun onReceive(context: Context?, intent: Intent?) {
            val manager = context?.getSystemService(CONNECTIVITY_SERVICE) 
                    as ConnectivityManager
            val oldType = intent?.getIntExtra(EXTRA_NETWORK_TYPE, TYPE_NONE)
            var newType = manager.activeNetworkInfo.type

            newType = when {
                oldType == TYPE_MOBILE && newType == TYPE_WIFI -> TYPE_NONE
                oldType == TYPE_WIFI && newType == TYPE_MOBILE -> TYPE_NONE
                else -> newType
            }

            if (lastType == newType) {
                return
            }

            listener?.invoke(lastType, newType)
        }
    }
}
```

定义网络变化监听器。

```kotlin
package com.samelody.samples.lifecycle

/**
 * The network type changed listener. Called when the network type is changed.
 *
 * @author Belin Wu
 */
typealias OnNetworkChangedListener = (Int, Int) -> Unit
```

在 `Activity` 中使用该观察者。

```kotlin
package com.samelody.samples.lifecycle

import android.os.Bundle
import android.util.Log
import androidx.appcompat.app.AppCompatActivity

/**
 * The sample activity.
 *
 * @author Belin Wu
 */
class SampleActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val observer = NetworkObserver(this)
        observer.listener = { from: Int, to: Int ->
            Log.d("Sample", "The network is changed from $from to $to")
        }
        lifecycle.addObserver(observer)
    }
}
```

在 `Service` 中使用该观察者。

```kotlin
package com.samelody.samples.lifecycle

import android.util.Log
import androidx.lifecycle.LifecycleService

/**
 * The sample service.
 *
 * @author Belin Wu
 */
class SampleService : LifecycleService() {

    override fun onCreate() {
        super.onCreate()
        val observer = NetworkObserver(this)
        observer.listener = { from: Int, to: Int ->
            Log.d("Sample", "The network is changed from $from to $to")
        }
        lifecycle.addObserver(observer)
    }
}
```

## 参考资料

- [Handling lifecycles with lifecycle-aware components](https://developer.android.google.cn/topic/libraries/architecture/lifecycle)
- [Adding Components to your Project](https://developer.android.google.cn/topic/libraries/architecture/adding-components)
- [Book: Android's Architecture Components
](https://commonsware.com/AndroidArch/)