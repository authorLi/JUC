# JUC源码之Future

## Future接口

这里也来看一下Future类的代码，与ForwardingFuture类不同，Future接口是JUC包下的，也就是说它是JDK自带的接口

### 类简介

```java
public interface Future<V> {
```

- Future类代表了一个异步计算的结果。
- 它的方法用于检验`计算是否完成`、`等待计算完成`和`检索计算结果`。
- 它的计算结果只能在计算完成后通过`get()`方法获取，必要时它会阻塞住，直到它计算完成准备就绪。
- 可以使用`cancel()`方法来进行取消计算的操作。
- 它还额外提供了方法来查询异步计算是**正常计算结束**还是**被取消**。
- 如果只是想提供一个可以被取消的方法而不在意其返回的计算结果来使用Future，可以使用Future的泛型”Future<?>“并在结束后返回”null“作为标志即可。

### 小例子

```java
interface ArchiveSearcher { String search(String target); }
class App {
  ExecutorService executor = ...
  ArchiveSearcher searcher = ...
  void showSearch(final String target)
      throws InterruptedException {
    Future<String> future
      = executor.submit(new Callable<String>() {
        public String call() {
            return searcher.search(target);
        }});
    displayOtherThings(); // do other things while searching
    try {
      displayText(future.get()); // use future
    } catch (ExecutionException ex) { cleanup(); return; }
  }
}}
```

### 方法

##### cancel()

```java
boolean cancel(boolean mayInterruptIfRunning);
```

- **尝试**去取消异步计算，如果**计算已经完成**或**计算已经被取消**或**优于其他原因无法被取消**，则此尝试会失败。
- 如果**已经成功**或者**调用此方法之前，异步计算还未开始**，那么异步计算将永远不会开始
- 如果异步计算已经开始，那么传入的参数`mayInterruptIfRunning`将决定是否应当中断执行此任务的线程以尝试停止该异步计算
- 此方法返回后，再调用`isDone()`方法将会永远返回true来表示任务完成
- 如果此方法返回true，那么在此方法执行结束后再调用`isCancelled()`方法将永远返回true

##### isCancelled()

```java
boolean isCancelled();
```

- 如果在异步计算正常完成之前此任务被取消，则返回true

##### isDone()

```java
boolean isDone();
```

- 当任务完成时会返回true
- 任务完成指的是：
  - 异步计算正常完成
  - 出现异常导致任务退出
  - 任务被取消

##### get()

```java
V get() throws InterruptedException, ExecutionException;
```

- 如果有必要，那么等待异步计算完成并检索其结果
- 可能会抛出以下三种异常：
  - CancellationException：当异步计算被取消时抛出
  - ExecutionException：当异步计算过程中出错抛出
  - InterruptedException：当执行异步计算的线程被中断时抛出

##### get(long timeout, @NotNull TimeUnit unit)

- 如果有必要，最多等待给定的时间以完成异步计算并检索其值(如果返回值可用)
- 可能会抛出以下四种异常：
  - CancellationException：当异步计算被取消时抛出
  - ExecutionException：当异步计算过程中出错抛出
  - InterruptedException：当执行异步计算的线程被中断时抛出
  - TimeoutException：如果等待超过指定时间时抛出



## RunnableFuture接口

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * 运行异步计算，除非它已经被取消
     */
    void run();
}
```

此子接口不仅继承了Future，而且还继承了Runnable接口。接口内部定义了`run()`方法来进行异步计算，除非任务被取消。



## ScheduledFuture接口

```java
public interface ScheduledFuture<V> extends Delayed, Future<V> {
}
```

继承了`Delayed`接口和`Future`接口，然而他却没有任何方法。说明此接口是将Delayed接口和Future接口的功能整合起来了，这是一个可以查看延迟的Future接口实现。



## Delayed接口

```java
public interface Delayed extends Comparable<Delayed> {

    /**
     * 返回以给定的时间为止所剩余的延迟时间；如果返回零或者负数则代表延迟已经结束
     */
    long getDelay(TimeUnit unit);
}
```

- 这个接口是一种混合风格的接口，它用来标记一些在给定延迟后需要对其进行操作的对象。
- 此接口的实现必须定义一个`compareTo()`方法并且此方法必须与`getDelay()`方法保持一致的顺序



## RunnableScheduledFuture接口

```java
public interface RunnableScheduledFuture<V> extends RunnableFuture<V>, ScheduledFuture<V> {

    /**
     * 如果任务是周期性的，此方法将返回true，非周期性则返回false
     */
    boolean isPeriodic();
}
```

- 非常直接的可以看到这个接口实现了`ScheduledFuture`接口(getDelay的功能)和`RunnableFuture`接口(run的功能)的功能



## FutureTask类

### 类简介

```java
public class FutureTask<V> implements RunnableFuture<V> {
```

- 此类实现了`RunnableFuture`接口
- 则是一个可取消的异步计算类，此类提供了一个Future接口的基本实现，并且提供了开始和取消计算、查询计算是否完成和检索计算结果的方法(这些方法都是实现了Future的方法)。
- 仅仅当计算完成后才可以去检索计算结果
- 如果计算没有完成，`get()`方法将会阻塞
- 一旦计算完成，那么此次计算不能够被重启或者取消(除非调用`runAndReset()方法`)
- FutureTask可以被用于包装`Runnable`或者`Callable`对象。因为FutureTask实现了Runnable接口，所以它可以被提交到一个Executor执行器去执行
- 除了作为一个独立的类，此类提供了一些在自定义任务时可能用得到的受保护的方法

### 变量

```java
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;

/** 底层的callable对象，在执行后可能被置为空 */
private Callable<V> callable;
/** get()方法返回的返回值，或是get()方法抛出的异常 */
private Object outcome; // non-volatile, protected by state reads/writes
/** 运行callable对象的线程，volatile修饰 */
private volatile Thread runner;
/** 存储了所有等待线程的链表节点 */
private volatile WaitNode waiters;

static final class WaitNode {
  	/** 存储当前线程 */
    volatile Thread thread;
  	/** 存储下一个等待的线程 */
    volatile WaitNode next;
    WaitNode() { thread = Thread.currentThread(); }
}

//Unsafe相关
    private static final sun.misc.Unsafe UNSAFE;
    private static final long stateOffset;
    private static final long runnerOffset;
    private static final long waitersOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = FutureTask.class;
            stateOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("state"));
            runnerOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("runner"));
            waitersOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("waiters"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
```

- 变量state使用`volatile`关键字修饰
- 前面几个变量代表了FutureTask的运行状态。最一开始是新建状态(NEW)；仅会在`set()`方法或者`setException()`方法或者`cancel()`方法中才会由运行状态转变为终止状态。在执行过程中，可能变为**COMPLETING**(当产生了计算结果)或者**INTERRUPTING**(仅当取消了任务的执行，中断了运行中的程序)
- 可能存在的状态转换：
  - NEW -> COMPLETING -> NORMAL
  - NEW -> COMPLETING -> EXCEPTIONAL
  - NEW -> CANCELLED
  - NEW -> INTERRUPTING -> INTERRUPTED

### 构造器

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;
}
```

- 此构造器会去执行给定的Callable对象，并且把state状态定为NEW(新建)

```java
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;
}
```

- 此方法会去执行给定的Runnable对象，当运行成功后将返回给定的结果值。

- 它的内部调用的是`Executors`工具类的`callable()`方法

- 在执行器内部又是由`RunnableAdapter`类实现的，它重写了`call()`方法，在运行了任务之后，返回了给定的返回值

  ```java
  public T call() {
      task.run();
      return result;
  }
  ```

