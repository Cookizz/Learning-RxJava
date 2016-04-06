# RxJava源码解析

## Java闭包

Java通过接口实现函数闭包。

	interface Map<T, R> {
    	R call(T input);
	}

	// 一个求平方的函数闭包
	Map square = new Map<Integer, Integer>() {
    	@Override
    	public Integer call(Integer input) {
        	return input * input;
    	}
	};

通常，你需要一个高阶函数来使用函数闭包
    

	int intMap(Map<Integer, Integer> map, int input) {
    	return map.call(input);
	}


传入闭包参数求平方
    
	int nine = intMap(square, 3);


RxJava中有2种闭包：**Action**和**Func**，Action是Rx链的端点，即最开始和最末端的闭包；Func即函数，是真正意义上的函数闭包。**OnSubscribe**和**Subscriber**属于Action闭包（虽然Subscriber不继承自Action），**Operator**属于Func闭包。

<br/>

## Rx调用模型
我们视野中的RxJava模型屏蔽了一切细节，看起来就是一系列操作按顺序连接。

	Observable.from(source)
        	.op1(func1)
    	    .op2(func2)
        	.op3(func3)
       		.subscribe(subscriber);


本质一点

	Observable.from(source)
    	    .lift(Op1)
   		    .lift(Op2)
        	.lift(Op3)
        	.subscribe(subscriber);

形象一点
    
    Source -> op1 -> op2 -> op3 -> LastOp

真实模型

	onSubscribe.parent.parent.parent
        	.call(op1(op2(op3(subscriber))));
            
**柯里化（Currying）**为一行语句

	compositOperator(subscriber)

### lift()的作用

	1. 返回一个新的Observable，内含一个新的OnSubscribe< R >。
	2. OnSubscribe< T >链表头部添加OnSubscribe< R >，作为	subscribe()的新起点。
	3. 使用Operator将新的Subscriber< R >还原为旧的Subscriber< T >，使其持有一个从T => R的客户实现的函数闭包。

### Operator的作用
	Operator是一个Subscriber => Subscriber的高阶函数闭包，它的职能是在Observable.subscribe()时将当前Subscriber< R >打回原形Subscriber< T >，当Subscriber< T >被invoke时，将传入Operator的客户闭包T => R作用于传入的T，返回R后invoke Subscribe< R >。

### 案例1：map()分析
### 案例2：flatMap()分析

  <br/>
  
## Rx调度模型
RxJava的线程调度使用多种类别的ExecutorService实现，其中对应关系如下。

Schedulers.io() -> newCachedThreadPool()
Schedulers.computation() -> newScheduledThreadPool()
Schedulers.newThread() -> newSingleThreadExecutor()

上述3种调度方式所创建的线程都是守护线程。

	Executors.newXxxThreadPool(new ThreadFactory() {
    	@Override
    	public Thread newThread(Runnable r) {
        	Thread t = new Thread(r, "RxXxxThreadPool-" + counter.incrementAndGet());
        	t.setDaemon(true); // 守护线程
        	return t;
    	}
	})；

### ObserveOnOperator
当我们调用observeOn(Schedulers.xxx())时，ObserveOnOperator为我们创建了一个消息队列:

	ConcurrentLinkedQueue<Object> queue =
        	new ConcurrentLinkedQueue<Object>();

当触发onNext/onError/onComplete时，传入的数据会添加到队列中（若没有传入的数据或传入的数据为空，比如onComplete、onNext(null)，则添加一个自定义哨兵），并通知Scheduler进行队列的调度。

	@Override
	public void onNext(final T t) {
    	queue.offer(on.next(t));
    	schedule();
	}

	@Override
	public void onCompleted() {
    	queue.offer(on.completed());
    	schedule();
	}

	@Override
	public void onError(final Throwable e) {
    	queue.offer(on.error(e));
    	schedule();
	}

当队列不为空时，一次性消费掉队列中的数据和哨兵：

	private void pollQueue() {
    	do {
        	Object v = queue.poll();
        	on.accept(observer, v);
    	} while (counter.decrementAndGet() > 0);
	}
    
### SubscribeOnOperator
subscribeOn()与observeOn()有本质的区别，它包含组合操作：nest + OperatorSubscribeOn。

	public final Observable<T> subscribeOn(Scheduler scheduler) {
    	return nest().lift(new OperatorSubscribeOn<T>(scheduler));
	}

nest()将当前Observable作为一项数据，返回一个发射它的新Observable。

	public final Observable<Observable<T>> nest() {
    	return just(this);
	}

OperatorSubscribeOn所做的唯一一件重要事情就是：将当前环节的onSubscribe.call()动作放在它所指定的线程中。

	@Override
	public void onNext(final Observable<T> o) {
    	subscriber.add(scheduler.schedule(new Action1<Inner>() {
        	@Override
        	public void call(final Inner inner) {
         	   o.unsafeSubscribe(subscriber);
        	}
    	}));
	}

所以毫无疑问，当一条Rx链中有多个subscribeOn()，看起来起作用的总是第一个。

	Observable.just("a")
        	.subscribeOn(Schedulers.io()) // 第一句onSubscribe才会影响到doOnNext
        	.doOnNext(firstAction)
        	.subscribeOn(Schedulers.computation())
        	.doOnNext(secondAction)
        	.observeOn(Schedulers.newThread())
        	.doOnNext(thridAction)
        	.observeOn(AndroidSchedulers.mainThread())
        	.doOnNext(fourthAction)
        	.subscribe();
