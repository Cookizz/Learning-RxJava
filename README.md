# ![Rx_Logo_S](http://7xawtr.com1.z0.glb.clouddn.com/Rx_Logo_S.png) RxJava学习笔记

RxJava是ReactiveX的Java语言版本实现。

ReactiveX是一种响应式编程模型，全称：Reactive Extensions，由微软架构师领导的团队开发，并在2012年11月开源。目标是提供一致的编程接口，帮助开发者更方便地处理异步数据流，从而构建**响应式应用**。

### ReactiveX概述
##### ReactiveX是一种FRP
FRP：Functional Reactive Programming。
FRP是基于观察者模式的一种编程范式（笔者个人习惯称它为“通用范式”，与“重复过程”并列，二者都属于“轮子”）。
使用FRP就基本上是在面向异步事件流编程了，FRP编程模型具有以下优势：
- 函数式风格：对可观察数据流使用无副作用的输入输出函数，避免了程序里错综复杂的状态。
- 简化代码：Rx的操作符通通常可以将复杂的难题简化为很少的几行代码。
- 异步错误处理：传统的try/catch没办法处理异步计算，Rx提供了合适的错误处理机制。
- 轻松使用并发：Rx的Observables和Schedulers让开发者可以摆脱底层的线程同步和各种并发问题。

但由于：
+ 很多项目复杂度没有达到使用FRP的地步。
+ 大多数开发者不理解FRP的抽象层次。

导致FRP并没有像OOP那样路人皆知。

##### ReactiveX用于构建响应式应用
Bruce Eckel（著有多部编程书籍）和Jonas Boner（Akka的缔造者和Typesafe的CTO）发表了“响应式宣言”，在其中尝试着定义什么是响应式应用。

这样的应用应该能够：
- 对事件做出响应：事件驱动的本质，让响应式应用能够支持文中提到的若干特性。
- 对负载做出响应：聚焦于可扩展性，而不是单用户性能。
- 对失败做出响应：建立弹性系统，能够从各个层级进行恢复。
- 对用户做出响应：综合上述特征，实现交互式用户体验。

##### 多语言版本
Rx在近几年逐渐流行起来，现在已经支持了大部分主流编程语言，比较流行的有RxJava、RxJS、ReactiveCocoa，另外，ReactiveX的Swift语言版本RxSwift现也在Github逐渐地完善起来。

本节将要介绍的RxJava是ReactiveX的Java语言实现版本，着重讲解其在Android开发中的使用场景。

### RxJava基础理论
##### 观察者模式
![观察者模式](http://7xn2zf.com1.z0.glb.clouddn.com/rxjavarxjava_2.png)

RxJava的观察者模式用图形表示如下：

![RxJava的观察者模式](http://7xn2zf.com1.z0.glb.clouddn.com/rxjavarxjava_3.png)

也可以使用代码来表达：

```java
// 创建被观察者
Observable<String> observable =
    Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("item 1");
                subscriber.onNext("item 2");
                subscriber.onNext("item 3");
                subscriber.onCompleted();
            }
        });
// 创建观察者，并订阅observable
observable.subscribe(new Observer<String>() {
    @Override
    public void onCompleted() {
        Log.d(TAG, "Completed!");
    }

    @Override
    public void onError(Throwable e) {
        Log.d(TAG, "Error!");
    }

    @Override
    public void onNext(String s) {
        Log.d(TAG, "Item: " + s);
    }
});
```

##### 创建事件流
![事件序列](http://7xawtr.com1.z0.glb.clouddn.com/eventsequence.png)

RxJava通过创建事件流来处理数据，事件流可以通过以下途径创建：
- 代码调用

        // create()是RxJava产生事件源的根方法
        // 所有其它创建方式的内部实现都是create
        Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("item 1");
                subscriber.onNext("item 2");
                subscriber.onNext("item 3");
                subscriber.onCompleted();
            }
        })
- 数组

        Observable.from(new String[] {"Rx", "Ja", "va"});
- Iterable

        // 线性表
        Observable.from(new ArrayList<String>());
        // 集合
        Observable.from(new HashSet<String>());
- Callable
        
        // Callable对象
        Observable.fromCallable(new Callable<String>() {
            @Override
            public String call() throws Exception {
                return "Hello Rx!";
            }
        });
- Future

        // Future对象
        Observable.from(Executors.newFixedThreadPool(3)
                .submit(new FutureTask<>(new Callable<String>() {
            @Override
            public String call() throws Exception {
                return "Hello Rx!";
            }
        })));
        
- 代码写死（一般用来测试）

        Observable.just("Rx", "Ja", "va");
以上方法都会返回一个Observable对象，在Java 8中管它叫Stream，具有以下特性：
- 时序性

        Observable以时间顺序产生事件，发出事件的时间间隔可以是CPU时钟级别，也可以是毫秒、秒或者更久。
- 不可变性

        Observable数据源对象一旦创建，其包含（发出）的数据是不可变的，所有操作符均无法改变它的状态，因而数据源可以被多次重复使用。

##### 操作符
在响应式编程中，对数据流的处理基于操作符的链式组合来实现，操作符本身是一个高阶函数，每一个操作符接受一个或多个**函数闭包**（Java语言通过对象实现函数闭包）作为参数，返回一个特定函数。

Rx的操作符具有以下特性：

- 链式

        Observable.from(new String[] {"Rx", "Ja", "va", "Rx", "va"})
                .distinct()
                .reduce(new Func2<String, String, String>() {
                    @Override
                    public String call(String s, String s2) {
                        return s.concat(s2);
                    }
                })
                .subscribe(new Action1<String>() {
                    @Override
                    public void call(String s) {
                        Log.d(TAG, "I'm learning " + s);
                    }
                });
        
- 声明式（惰性计算）

        声明在Observable上的所有操作符不会做任何事情，除非：
        1. Observable的数据源开始产生数据。
        2. Observable被观察了。
- 操作符可任意指定工作线程

        Observable.just(url)
                .map(new Func1<URL, String>() {
                    @Override
                    public String call(URL url) {
                        JSONObject json = getMsgFromServer(url);
                        return json.optString("msg");
                    }
                })
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<String>() {
                    @Override
                    public void call(String s) {
                        mTxtMsg.setText(s);
                    }
                });

### 案例剖析
##### Case 1：躺着解决“迷之缩进”
使用命令式编程方式存在一个致命的缺陷：随着代码逻辑嵌套层次越来越深，缩进会越来越严重，这对代码可读性和可维护性而言是致命的。

使用RxJava可以轻松解决此问题，无论要处理的逻辑多么复杂，响应式编写出来的代码依然可以保持简洁。场景：遍历磁盘目录，获取所有png图片并显示在界面上。

常规思路：
```java
new Thread() {
    @Override
    public void run() {
        super.run();
        for (File folder : folders) {
            File[] files = folder.listFiles();
            for (File file : files) {
                if (file.getName().endsWith(".png")) {
                    final Bitmap bitmap = getBitmapFromFile(file);
                    getActivity().runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            imageCollectorView.addImage(bitmap);
                        }
                    });
                }
            }
        }
    }
}.start();
```
RxJava实现：
```java
Observable.from(folders)
.flatMap(new Func1<File, Observable<File>>() {
    @Override
    public Observable<File> call(File file) {
        return Observable.from(file.listFiles());
    }
})
.filter(new Func1<File, Boolean>() {
    @Override
    public Boolean call(File file) {
        return file.getName().endsWith(".png");
    }
})
.map(new Func1<File, Bitmap>() {
    @Override
    public Bitmap call(File file) {
        return getBitmapFromFile(file);
    }
})
.subscribeOn(Schedulers.io())
.observeOn(AndroidSchedulers.mainThread())
.subscribe(new Action1<Bitmap>() {
    @Override
    public void call(Bitmap bitmap) {
        imageCollectorView.addImage(bitmap);
    }
});
```

##### Case 2：消息推送SDK订阅过程
先检查订阅缓存，若不存在，则请求网络：
```java
Observable.just("subscribe")
        .filter(new Func1<String, Boolean>() { // 判断是否有缓存
            @Override
            public Boolean call(String s) {
                String cache = getSubscribeCache();
                return TextUtils.isEmpty(cache);
            }
        })
        .map(new Func1<String, String>() { // 若无缓存，则请求网络
            @Override
            public String call(String s) {
                JSONObject json = subscribePubFromNetwork();
                if (json == null || json.isNull("openId")) {
                    throw new IllegalArgumentException("openId not found.");
                }
                return json.optString("openId");
            }
        })
        .subscribeOn(Schedulers.newThread()) // 指定新线程
        .subscribe(new Subscriber<String>() { // 处理订阅结果
            @Override
            public void onCompleted() {}

            @Override
            public void onError(Throwable e) {
                // TODO 处理请求错误
            }

            @Override
            public void onNext(String s) {
                // TODO 获取openId
            }
        });
```

##### Case 3：RxJava & Retrofit

平常我们是这样写Retrofit接口的：
```java
@GET("/user/{id}/photo")
void getPhoto(@Path("id") int id, Callback<Photo> cb);
```
其实还可以这么写：
```java
@GET("/user/{id}/photo")
Observable<Photo> getPhoto(@Path("id") int id);
```
没错，返回的就是RxJava的Observable数据源对象，拿到了它，不管你怎么蹂躏都可以。


##### Case 4：Android按钮防“抖”
害怕按钮连点2次怎么办？使用RxBinding：
```gradle
dependencies {
    compile 'com.jakewharton.rxbinding:rxbinding:0.3.0'
}
```
```java
RxView.clicks(button) // 劫持button的onClick事件
.throttleFirst(200, TimeUnit.MILLISECONDS) // 设置200毫秒之内无法二次点击
.subscribe(subscriber); // 监听点击事件
```

### 响应式进阶
##### 操作符实现原理：lift代理（我喜欢叫它“进化"）
纵观RxJava Observable的所有非创建型操作符的源码，可以发现，基本上所有操作符都是在调用lift(Operator op)方法：
```java
public final Observable<T> distinct() {
    return lift(OperatorDistinct.<T> instance());
}
public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
    return lift(new OperatorMap<T, R>(func));
}
```

不难看出，RxJava实现了各种各样的Operator，Operator的各种实现才是真正意义上的操作符。
下面是lift方法的核心代码（砍掉额外的异常捕获、钩子等）：
```java
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
    return new Observable<R>(new OnSubscribe<R>() {
        @Override
        public void call(Subscriber<? super R> o) {
            // 当新Observable被观察时
            // 使用新操作符创建旧Subscriber的代理，等待旧Subscriber的成功冒泡返回
            Subscriber<? super T> st = operator.call(o);
            // 新Observable持有旧Observable的调度器对象
            // 当新Observable被观察时，会通知旧Observable：“我要观察你了”
            onSubscribe.call(st);
        }
    });
}
```

可以用一个比较贴切的词去描述lift所做的事情：进化。

当一个Observable对象被lift()之后，会返回一个进化了的新Observable，这个Observable持有刚才传入lift()的操作符的所有权以及旧Observable的观察权，并对外声明：自己是一个新的Observable，自己观察了被lift之前的Observable，如果谁想观察我，我会用我的操作符对旧Observable的原始数据做一次处理，再把处理过的数据通知给你。

![lift机理](http://7xawtr.com1.z0.glb.clouddn.com/lift_mechanism.png)

##### 自定义操作符
如果能掌握lift代理的秘密，我们便可以照葫芦画瓢实现自己想要的操作符了。
```java
public final class MyOperator<T, U> implements Operator<T, T> {
    
    private Func1 mTransformer;
    /**
     * 如果需要传入闭包进行数据变换，则需要为构造函数传入Function对象
     * 这里用Func1举例
     */
    public MyOperator(Func1<? super T, ? extends R> transformer) {
        // 如果有Transformer，则初始化
        mTransformer = transformer;
    }
    
    @Override
    public Subscriber<? super T> call(final Subscriber<? super T> child) {
        return new Subscriber<T>(o) {
            @Override
            public void onCompleted() {
                o.onCompleted();
            }
            
            @Override
            public void onError(Throwable e) {
                o.onError(e);
            }

            @Override
            public void onNext(T t) {
                try {
                    // map变换
                    o.onNext(transformer.call(t));
                } catch (Throwable e) {
                    Exceptions.throwOrReport(e, this, t);
                }
            }
        };
    }
}
```
定义好操作符之后，便可以通过显式调用Observable的lift完成操作符的变换效果：
```java
Observable.just("My", "Ope", "rator")
        .lift(new MyOperator<String, Integer>(func));
```

### 附录
##### 进阶之路
- 如果对RxJava的概念仍然模糊，则推荐国人写的一篇博客：
    [给Android开发者写的RxJava详解](http://gank.io/post/560e15be2dca930e00da1083#toc_28)。
- 如果对RxJava有了一定的了解并能够写一些Demo，强烈推荐阅读[官方文档](http://reactivex.io)，着重加强对RxJava丰富的操作符的掌握。
- 如果掌握了RxJava的API和原理，则建议查看[Github](https://github.com/ReactiveX/RxJava)仓库，也许你能够为它贡献Pull Requests。

##### 关于本文
- 本文基于Markdown标记语言制作，使用了一款非常好用的在线Markdown工具：[dillinger](http://dillinger.io/)，可以随时在线保存以及导出HTML、Markdown源文件以及PDF格式。
- 3处图片资源和Case 1示例代码资源引用了[扔物线的RxJava博客](http://gank.io/post/560e15be2dca930e00da1083#toc_28)。
- 其他示例图片皆来自[ReactiveX官方文档](http://reactivex.io)。
- 概述的部分内容引用了[新兴趋势：反应性编程](http://www.infoq.com/cn/news/2013/08/reactive-programming-emerging/)。
- 其他内容皆原创，转载须经本人同意，笔者邮箱：com.cookizz@gmail.com。
