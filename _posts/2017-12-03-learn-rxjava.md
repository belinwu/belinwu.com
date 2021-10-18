---
layout: post
title: 初学RxJava
date: 2017-12-03 23:34
tags: RxJava Java
categories: 编程
---

## 什么是RxJava？

这个名词包含两部分：

- Rx(是ReactiveX、Reactive Extensions、Reactive Programming的简称)：An API for asynchronous programming with observable streams.
- Java: Rx在JVM上实现。

在RxJava的GitHub里官方给出了这样的示意：

> A library for composing asynchronous and event-based programs by using observable sequences.

Rx组合了`观察者模式`、`迭代器模式`和`函数式编程范式`思想，所以有时候也将这种编程方法叫做`Functional Reactive Programming`。

## Hello world

```java
Observable.just("Hello", "world!")
	    .subscribe(new Consumer<String>() {
	        @Override
	        public void accept(String item) throws Exception {
	            System.out.print(item + " ");
	        }
	    });
// Hello world!
```
`Observable.just()`将传入的2个字符串项转化成一个Reactive Stream，调用`subscribe()`订阅方法观察该Stream里的数据项。图示如下：

![image.png](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e4f6706f576c~tplv-t2oaga2asx-image.image)


## Observable和Observer

`Observable`是一种基于事件推送并且可组合的迭代器，它可以推送3种事件：

- `onNext()`：接收推送过来的数据项。
- `onError()`：接收失败事件。
- `onComplete()`：响应流推送已完成。

这3种事件与`Observer`类中的抽象方法一一对应。

```java
public interface Observer<T> {
    void onNext(@NonNull T t);
    void onError(@NonNull Throwable e);
    void onComplete();
}
```
使用`Observer`订阅`Observable`：

```java
Observable.just("Hello", "world!")
        .subscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {

            }

            @Override
            public void onNext(String item) {
                out.print(item + " ");
            }

            @Override
            public void onError(Throwable e) {
                out.println("error: " + e.getMessage());
            }

            @Override
            public void onComplete() {
                out.println("done");
            }
        });
```

`Observer.onSubscribe()`方法的入参`Disposable`可用来取消订阅。下面使用RxJava提供的`DisposableObserver`类演示取消订阅功能。

```java
Disposable disposable = Observable.just("Hello", "world!")
        .subscribeWith(new DisposableObserver<String>() {
            @Override
            public void onNext(String item) {
                out.print(item + " ");
            }

            @Override
            public void onError(Throwable e) {
                out.println("error: " + e.getMessage());
            }

            @Override
            public void onComplete() {
                out.println("done");
            }
        });

// 在适当的时机取消订阅
disposable.dispose();
```

## Operator

`Observable.just()`也是一种Operator，它将传入的数据项转化成了响应流。对响应流应用一些Operator可以进行函数式编程、数据转换、过滤等许多功能。

### map
![image.png](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e4f6708a0336~tplv-t2oaga2asx-image.image)


```java
Observable.just("Hello", "world!")
        .map(new Function<String, Integer>() {
            @Override
            public Integer apply(String item) throws Exception {
                return item.length();
            }
        })
        .subscribe(new DisposableObserver<Integer>() {
            @Override
            public void onNext(Integer item) {
                out.print(item + " ");
            }

            @Override
            public void onError(Throwable e) {
                out.println("error: " + e.getMessage());
            }

            @Override
            public void onComplete() {
                out.println("done");
            }
        });
```

### create

使用`create()`方法可以自己动手创建响应流。

![image.png](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e4f6744684cf~tplv-t2oaga2asx-image.image)


```java
Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> e) throws Exception {
                try {
                    e.onNext("Hello");
                    e.onNext(" world!");
                    e.onComplete();
                } catch (Exception ex) {
                    e.onError(ex);
                }
            }
        })
        .subscribe(new DisposableObserver<String>() {
            @Override
            public void onNext(String item) {
                out.print(item + " ");
            }

            @Override
            public void onError(Throwable e) {
                out.println("error: " + e.getMessage());
            }

            @Override
            public void onComplete() {
                out.println("done");
            }
        }); 
```

## 使用场景

### 异步操作

作为一个Android开发者，我们都知道：不应该阻塞UI线程，应该将耗时的计算放在异步线程中。我们有多种实现方式，这里不再多做说明，直接使用RxJava实现异步操作。

```java
Observable.create(new ObservableOnSubscribe<String>() {
        @Override
        public void subscribe(ObservableEmitter<String> e) throws Exception {
            // 该方法将在IO线程里执行
            try {
                e.onNext("Hello");
                e.onNext(" world!");
                e.onComplete();
            } catch (Exception ex) {
                e.onError(ex);
            }
        }
    })
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new DisposableObserver<String>() {
        @Override
        public void onNext(String item) {
            // 在UI线程里执行
            out.print(item + " ");
        }

        @Override
        public void onError(Throwable e) {
            // 在UI线程里执行
            out.println("error: " + e.getMessage());
        }

        @Override
        public void onComplete() {
            // 在UI线程里执行
            out.println("done");
        }
    });
```

### 多异步操作

现在要实现一个修改头像的需求，分成以下3个步骤来完成：

- 获取图片上传token；
- 使用上传token上传图片；
- 调用修改个人信息接口修改头像信息。

很显然这3个操作都需要在异步线程里进行，而且是按顺序一一进行的。通常使用回掉来实现非阻塞异步操作，但随着连续多个异步操作容易引入Callback Hell问题。

使用RxJava的`flatMap()`方法可以优雅地解决这些问题。

```java
getUploadToken()
    .flatMap(token -> uploadImage(token))
    .flatMap(imageUrl -> updateUser(imageUrl))
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new DisposableObserver<Boolean>() {
        @Override
        public void onNext(Boolean item) {
            out.print(item + " ");
        }

        @Override
        public void onError(Throwable e) {
            out.println("error: " + e.getMessage());
        }

        @Override
        public void onComplete() {
            out.println("done");
        }
    });

public Observable<String> getUploadToken() {
    return Observable.create(e -> {
        try {
            String token = "get token from remote server";
            e.onNext(token);
            e.onComplete();
        } catch (Exception ex) {
            e.onError(ex);
        }
    });
}

public Observable<String> uploadImage(final String token) {
    return Observable.create(e -> {
        try {
            String imageUrl = "upload image to remote server with token: " + token;
            e.onNext(imageUrl);
            e.onComplete();
        } catch (Exception ex) {
            e.onError(ex);
        }
    });
}

public Observable<Boolean> updateUser(final String imageUrl) {
    return Observable.create(e -> {
        try {
            // update user with image url
            e.onNext(true);
            e.onComplete();
        } catch (Exception ex) {
            e.onError(ex);
        }
    });
}
```

![image.png](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e4f6745a72ab~tplv-t2oaga2asx-image.image)

### 防止连续点击

在Android开发中会遇到这样的问题：短时间内快速点击某个按钮，按钮的单击监听器会被多次调用，这样会引入一写异常情况。

同样可以使用RxJava的`throttleFirst()`方法来解决这种问题。

![image.png](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/24/1691e4f68021cd59~tplv-t2oaga2asx-image.image)

```java
RxView.clicks(findViewById(R.id.btn_throttle))
    .throttleFirst(1, TimeUnit.SECONDS)
    .subscribe(aVoid -> {
        System.out.println("click");
    });
```

## 参考资料

- [RxJava](https://github.com/ReactiveX/RxJava)
- [RxAndroid](https://github.com/ReactiveX/RxAndroid)
- [RxBinding](https://github.com/JakeWharton/RxBinding)
- [ReactiveX](http://reactivex.io)