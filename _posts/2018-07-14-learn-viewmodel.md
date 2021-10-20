---
layout: post
title: 初学Android架构组件之ViewModel
date: 2018-07-14 23:23
tags: ViewModel Jetpack
categories: Android
---

在 Android 中，`Activity` 和 `Fragment` 这类 UI 组件会被系统销毁或重建，未特殊处理的 UI 数据将会丢失。以往处理这类问题时，会使用 `onSaveInstanceState()` 保存 UI 数据，在 `onCreate()` 方法里恢复 UI 数据，但是数据的大小和类型有限制。

看看下面的2个问题：

- 对于因手机 *Configuration Changes* 而被系统重建的界面，为了呈现之前的 UI 状态，采用上述的方式来实现起来会显得过于繁琐、不优雅。
- 在实践 *Separation of Concerns* 准则时，UI 组件主要是负责显示 UI 数据、响应系统与视图事件，若再负责数据的加载和管理，会变得臃肿、不易复用。

本文的主角 `ViewModel` 可以很好地解决这些问题。注意：它不是用来代替 `onSaveInstanceState()`。

## ViewModel

`ViewModel` 抽象类被设计成专门存储和管理 UI 数据，它的定义很简单，类中只有一个空实现的 `onCleared()`。

## 自定义 ViewModel

在 app 模块的 `build.gradle` 文件里添加依赖。

```groovy
dependencies {
    implementation "androidx.lifecycle:lifecycle-extensions:2.0.0-beta01"
}
```

自定义一个 `ViewModel` 子类 `MainViewModel`，它维护一个 UI 状态：`loading`。

```kotlin
class MainViewModel : ViewModel() {

    /**
     * The flag indicates that the content is loading or not.
     */
    val loading = false
}
```

在 UI 组件的 `onCreate()` 生命周期里，先使用 `ViewModelProviders.of()` 方法获取当前 UI 作用域里的 `ViewModelProvider` 对象。再通过 `ViewModelProvider` 类提供的 `get()` 方法获取 `ViewModel` 对象。

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val model = ViewModelProviders.of(this).get(MainViewModel::class.java)
    }
}

```

## 生命周期

`ViewModel` 的生命周期与 UI 组件的生命周期相关联，如下图所示：

![ViewModel 生命周期（来自官网）](/assets/img/viewmodel-lifecycle.png)

当 `Activity` 被系统重建时，`ViewModel` 对象不会被销毁，新的 `Activity` 对象拿到的是同一个 `ViewModel` 对象。可以很方便的使用 `ViewModel` 里的 UI 数据将之前的 UI 状态呈现给用户。比如上面的例子，呈现页面加载状态。

```kotlin
progressBar.visibility = when (model.loading) {
    true -> VISIBLE
    false -> INVISIBLE
}
```

当 `ViewModel` 所在的 UI 组件被真正销毁时，它的 `onCleared()` 方法会被调用，可以覆盖该方法清理资源。

## AndroidViewModel

当自定义的 `ViewModel` 类中需要使用应用上下文 `Context` 时，可以选择继承 `AndroidViewModel` 类，该类定义了 `Application` 字段。

## 参考资料

- [ViewModel Overview](https://developer.android.com/topic/libraries/architecture/viewmodel)