---
title: AsyncTask
date: 2019-02-19 14:16:00
categories:
- Android
- Thread
-
tags:
- Android
- Thread
---
# AsyncTask

## Abstract
```
public abstract class AsyncTask<Params, Progress, Result> {...}
```
AsyncTask is designed to be a helper class around {@link Thread} and {@link Handler} and does not constitute a generic threading framework.
AsyncTasks should ideally be used for short operations (a few seconds at the most.)
If you need to keep threads running for long periods of time, it is highly recommended you use the various APIs provided by the <code>java.util.concurrent</code> package such as {@link Executor}, {@link ThreadPoolExecutor} and {@link FutureTask}.</p>
<!-- more -->

## AsyncTask--Looper
UI线程：onPreExecute, publishProgress, onPostExecute的实现

### Constuctors
```java
public AsyncTask() {
   this((Looper) null);
}

private static Handler getMainHandler() {
    synchronized (AsyncTask.class) {
        if (sHandler == null) {
            sHandler = new InternalHandler(Looper.getMainLooper());
        }
        return sHandler;
    }
}

// Creates a new asynchronous task. This constructor must be invoked on the UI thread.
// @hide
public AsyncTask(@Nullable Looper callbackLooper) {
   mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
       ? getMainHandler()
       : new Handler(callbackLooper);//默认获取主线程Handler,亦可自定义HandlerThread.Looper
  .....
}
```

### 子线程返回主线程：
```java
Msg.what:
    private static final int MESSAGE_POST_RESULT = 0x1;
    private static final int MESSAGE_POST_PROGRESS = 0x2;
Msg.object:
private static class AsyncTaskResult<Data> {
   final AsyncTask mTask;
   final Data[] mData;

   AsyncTaskResult(AsyncTask task, Data... data) {
       mTask = task;
       mData = data;
   }
}

example:
Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                   new AsyncTaskResult<Result>(this, result));// result = doInBackground(mParams);
```


## AsyncTask--FutureTask
子线程：doInBackground
### Constructor
```java
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
    Params[] mParams;
}

public AsyncTask(@Nullable Looper callbackLooper) {
  mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);
            Result result = null;
            try {
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                result = doInBackground(mParams);///////.....
                Binder.flushPendingCommands();
            } catch (Throwable tr) {
                mCancelled.set(true);//AtomicBoolean: 取消任务标识
                throw tr;
            } finally {
                postResult(result);//////返回结果   sendMessage() --> UI线程
            }
            return result;
        }
    };

    ///Callable封装
    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get());////返回结果get()
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()",
                        e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);////任务中途取消
            }
        }
    };
}
```

---
## 运行
```java
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

SerialExecutor: sDefaultExecutor
方式一：
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
private volatile Status mStatus = Status.PENDING;

public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {//保证只能运行一次
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
方式二：
public static void execute(Runnable runnable) {
   sDefaultExecutor.execute(runnable); ///
}
```
### exec.execute(mFuture)&sDefaultExecutor.execute(runnable)
```
private static class SerialExecutor implements Executor {
   final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();//任务队列（双端队列）
   Runnable mActive;

   public synchronized void execute(final Runnable r) {
       mTasks.offer(new Runnable() {//insert into the tail
           public void run() {
               try {
                   r.run();
               } finally {
                   scheduleNext();//取出执行下一任务
               }
           }
       });
       if (mActive == null) {
           scheduleNext();
       }
   }

   protected synchronized void scheduleNext() {
       if ((mActive = mTasks.poll()) != null) {//任务队列中存在 head
           THREAD_POOL_EXECUTOR.execute(mActive);
           /*THREAD_POOL_EXECUTOR  -- ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            *    CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            *    sPoolWorkQueue, sThreadFactory);
            */
       }
   }
}
```

### FutureTask---运行
public class FutureTask<V> implements RunnableFuture<V>
public interface RunnableFuture<V> extends Runnable, Future<V>
```
//long RUNNER-->FutureTask中field:runner的地址偏移
public void run() {
   if (state != NEW ||
       !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))//Unsafe--CAS--内存比较
       //private volatile Thread runner;
       //判断runner是否为空若为空则赋值为当前线程并返回true    -- runner指向当前线程
       return;


   //新任务尚未执行，runner为空(赋值成功)--->
   try {
       Callable<V> c = callable;
       if (c != null && state == NEW) {
           V result;
           boolean ran;
           try {
               result = c.call();//执行Callale#call
               ran = true;
           } catch (Throwable ex) {
               result = null;
               ran = false;
               setException(ex);
           }
           if (ran)
               set(result);//
       }
   } finally {
       // runner must be non-null until state is settled to
       // prevent concurrent calls to run()
       runner = null;
       // state must be re-read after nulling runner to prevent
       // leaked interrupts
       int s = state;
       if (s >= INTERRUPTING)//中断，取消
           handlePossibleCancellationInterrupt(s);
   }
}

//正常结束任务
protected void set(V v) {
   if (U.compareAndSwapInt(this, STATE, NEW, COMPLETING)) {//新任务
       outcome = v;
       U.putOrderedInt(this, STATE, NORMAL); // final state --->状态设为NORMAL  正常结束
       finishCompletion();
   }
}

//Removes and signals all waiting threads, invokes done(), and nulls out callable
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (U.compareAndSwapObject(this, WAITERS, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }

    done();//调用重写方法done

    callable = null;        // to reduce footprint
}
```




-----
```
interface Future:提供cancel（任务取消）,    

A {@code Future} represents the result of an asynchronous  computation.  
Methods are provided to check if the computation is complete, to wait for its completion, and to retrieve the result of the computation.  

The result can only be retrieved using method {@code get} when the computation has completed, blocking if necessary until it is ready.  

Cancellation is performed by the {@code cancel} method.  
Additional methods are provided to determine if the task completed normally or was cancelled.

Once a computation has completed, the computation cannot be cancelled.


If you would like to use a {@code Future} for the sake of cancellability but not provide a usable result, you can declare types of the form {@code Future<?>} and return {@code null} as a result of the underlying task.
```

---
## FutureTask--取消任务
```java
public boolean cancel(boolean mayInterruptIfRunning) {
    if (!(state == NEW &&
          U.compareAndSwapInt(this, STATE, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))//不允许取消的任务：刚刚创建，尚未执行；     --将状态设置为INTERRUPTING or CANCELLED失败
        return false;
    try {    // in case call to interrupt throws exception
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;//runner指向当前线程
                if (t != null)
                    t.interrupt();//。。。
            } finally { // final state
                U.putOrderedInt(this, STATE, INTERRUPTED);
            }
        }
    } finally {
        finishCompletion();//结束
    }
    return true;
}
```
