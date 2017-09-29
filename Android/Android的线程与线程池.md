## Java并发的相关类

### Runnable

Runnable只有一个run方法，此方法没有返回值。

```java
public interface Runnable {  
    public abstract void run();  
}  
```

### Callable

Callable中有一个call()函数，**但是call()函数有返回值**。

```java
public interface Callable<V> {  
    V call() throws Exception;  
}  
```

### **Future**

Executor就是Runnable和Callable的调度容器，**Future就是对于具体的Runnable或者Callable任务的执行结果进行取消、查询是否完成、获取结果、设置结果操作**。

```java
public interface Future<V> {  
    boolean cancel(boolean mayInterruptIfRunning);  
    boolean isCancelled();  
    boolean isDone();  
    V get() throws InterruptedException, ExecutionException;  
    V get(long timeout, TimeUnit unit)  
        throws InterruptedException, ExecutionException, TimeoutException;  
}  
```

### FutureTask

FutureTask则是一个`RunnableFuture<V>`，而RunnableFuture实现了Runnbale又实现了`Futrue<V>`这两个接口。

```java
public class FutureTask<V> implements RunnableFuture<V>  
public interface RunnableFuture<V> extends Runnable, Future<V>
```

因此FutureTask可以提交给ExecuteService来执行，还可以直接通过get()函数获取执行结果，该函数会阻塞，直到结果返回。



## 线程池

ThreadPoolExecutor是Executor的实现，是线程池的真正实现。

1. FixedThreadPool：线程数量固定。
2. CachedThreadPool 线程数量不定，空闲线程有超时时间。
3. ScheduledThreadPool  核心线程数固定，非核心线程数无限制，非核心线程闲置会立即回收。
4. SingleThreadExecutor 只有一个核心线程，所有任务都在同一个线程中按顺序执行。

## AsyncTask

```java
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
	public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }
        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }
        mStatus = Status.RUNNING;
        onPreExecute();
        mWorker.mParams = params;
        exec.execute(mFuture);
        return this;
    }
```

sDefaultExecutor其实是一个串行的线程池，从SerialExecutor的execute方法可以看出。AsyncTask有两个线程池(SerialExecutor&THREAD_POOL_EXECUTOR)，其中SerialExecutor用于任务排队，而THREAD_POOL_EXECUTOR是真正执行任务的线程池。如果我们需要AsyncTask能并发执行，可以调用它的executeOnExecutor方法，传一个自定义的线程池即可。

executeOnExecutor方法内先调用onPreExecute方法，并将请求参数封装到mWorker中。

```java
    private final WorkerRunnable<Params, Result> mWorker;
    private final FutureTask<Result> mFuture;
    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                return postResult(doInBackground(mParams));
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occured while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```

mFuture 是FutureTask，mWorker是它的Callable。当mFuture被执行，执行内容是mWorker中的call方法。在call方法中，调用了doInBackground方法，并将返回结果通过postResult方法投递给sHandler来处理，sHandler将数据从子线程切换到主线程。

```java
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = sHandler.obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
    private static class InternalHandler extends Handler {
        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult result = (AsyncTaskResult) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```

**为什么默认AsyncTask不并行执行呢**

如果同时有几个AsyncTask一起并行执行的话，恰好AysncTask的使用者在`doInbackgroud`里面访问了相同的资源，但是自己没有处理同步问题；那么就有可能导致灾难性的后果！因此，API的设计者为了避免这种无意中访问并发资源的问题，干脆把这个API设置为默认所有串行执行的了。如果你明确知道自己需要并行处理任务，那么你需要使用`executeOnExecutor(Executor exec,Params... params)`这个函数来指定你用来执行任务的线程池，同时为自己的行为负责。

## HandlerThread

Handy class for starting a new thread that has a looper. The looper can then be used to create handler classes. Note that start() must still be called.


```java
    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```



## IntentService

内部利用一个HandlerThread，将Handler绑定到新的线程里面。onHandleIntent在Handler的handleMessage方法中执行，也就是在新线程中执行。

```java
    @Override
    public void onCreate() {
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();
        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }
        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
```

