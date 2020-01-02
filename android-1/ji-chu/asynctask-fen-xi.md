# AsyncTask 分析



> AsyncTask enables proper and easy use of the UI thread. This class allows you to perform background operations and publish results on the UI thread without having to manipulate threads and/or handlers.

AsyncTask 能够很方便的执行后台操作并在 UI 线程上发布结果，无需操作 Thread 和 Handler。 

### AsyncTask 基础

#### AsyncTask 使用

AsyncTask 是一个抽象类，需要继承它才能使用。子类至少覆写一个方法 `doInBackground(Params...)`，通常第二个覆写的方法为 `onPostExecute(Result)`，下边是一个简单示例：

```text
private class DownloadFilesTask extends AsyncTask<URL, Integer, Long> {
    protected Long doInBackground(URL... urls) {
        int count = urls.length;
        long totalSize = 0;
        for (int i = 0; i < count; i++) {
            totalSize += Downloader.downloadFile(urls[i]);
            publishProgress((int) ((i / (float) count) * 100));
            // Escape early if cancel() is called
            if (isCancelled()) break;
        }
        return totalSize;
    }

    protected void onProgressUpdate(Integer... progress) {
        setProgressPercent(progress[0]);
    }

    protected void onPostExecute(Long result) {
        showDialog("Downloaded " + result + " bytes");
    }
}
```

一旦创建，执行起来非常简单：

```text
new DownloadFilesTask().execute(url1, url2, url3);
```

需要注意的是，一个 AsyncTask 实例只能调用一次 `execute` 方法，并且此方法应该在 UI 线程被调用。

#### AsyncTask 的泛型参数

1. `Params`：异步任务所接收的参数类型，即 `doInBackground` 的参数。
2. `Progress`：异步任务计算过程中进度的单位，`onProgressUpdate` 的参数。
3. `Result`：异步任务的返回结果，`doInBackground` 的返回结果，`onPostExecute` 的参数。

这三个参数不是必须需要指定的，如果不需要，可以指定为 `Void`：

```text
private class MyTask extends AsyncTask<Void, Void, Void> { ... }
```

#### AsyncTask 覆写的四个方法

1. `onPreExecute()`：此方法会在异步任务执行之前在主线程被调用，通常用来设置任务，比如在 UI 线程显示进度条等。
2. `doInBackground(Params...)`：在 `onPreExecute()` 完成后在子线程中立即执行，用与执行耗时任务，执行结果会返回给 `onPostExecute(Result)` 进行处理。在这个方法中可以调用 `publishProgress(Progress...)` 将任务执行进度进行反馈到 UI 线程中的 `onProgressUpdate(Progress...)` 中进行处理。
3. `onProgressUpdate(Progress...)`：调用 `publishProgress(Progress...)` 之后将会在 UI 线程中执行此方法，可以对 UI 界面进行更新，如更新进度条等。
4. `onPostExecute(Result)`：子线程任务执行完成后将会在 UI 线程调用此方法，子线程的 `doInBackground(Params...)` 的结果会传递给它进行处理。

### AsyncTask 注意事项

#### 执行顺序

在初次引入 AsyncTask 时，以串行的方式执行任务，从 Android 1.6 开始，被更改为允许多个任务并行操作的线程池，从 Android 3.0 开始，任务开始串行执行，避免因并行造成的逻辑上的错误。

如果需要并行执行，可以使用如下方法：

```text
mTask.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR, params);
```

#### 使用场景

AsyncTask 应该被用作短时间的异步操作，如果需要长时间的异步操作，请使用 [Executor](https://developer.android.com/reference/java/util/concurrent/Executor.html), [ThreadPoolExecutor](https://developer.android.com/reference/java/util/concurrent/ThreadPoolExecutor.html) 和 [FutureTask](https://developer.android.com/reference/java/util/concurrent/FutureTask.html)。主要原因有两点：

* **与 Activity 的生命周期关联很差**：当 Activity 被销毁时，AsyncTask 未完成就不会被销毁，此时如果在 AsyncTask 中更新 UI，则会引起异常 `java.lang.IllegalArgumentException: View not attached to window manager`。
* **很容易引起内存泄漏**：当 AnsycTask 作为内部类时，如果异步任务执行时间过长，Activity 将不会被回收，从而引起内存泄漏。

### 源码分析（API 23）

AsyncTask 的执行从 `execute` 方法开始：

```text
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
```

在这个方法中调用了 `executeOnExecutor` 方法：

```text
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

`executeOnExecutor` 在一开始判断了当前 AsyncTask 的状态，如果不是处于 `PENDING` 则不能执行，当 AsyncTask 开始执行时会转换成 `RUNNING`，执行完成后会变成 `FINISHED`，由此可以看出一个 AsyncTask 实例只能执行一次。接下来便会调用 `onPreExecute()` 方法，由于 `execute` 是在 UI 线程执行的，所以 `onPreExecute()` 也在 UI 线程执行，能够对 UI 进行操作。下来会调用 `exec.execute(mFuture)`，exec 便是 AsyncTask 的静态变量 `sDefaultExecutor`，声明如下：

```text
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

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
```

`execute` 方法接收一个 Runnable 对象，这个对象为 mFuture，在构造函数中被初始化：

```text
public AsyncTask() {
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);

            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            //noinspection unchecked
            Result result = doInBackground(mParams);
            Binder.flushPendingCommands();
            return postResult(result);
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
                throw new RuntimeException("An error occurred while executing doInBackground()",
                        e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}
```

`execute` 中通过构造一个新的 Runnable 对象，在 `run` 方法中调用 mFuture 的 `run` 方法，然后将其加入 mTasks 队列中，紧接着判断当前活动任务是否为空，如果为空则执行 `scheduleNext`，``` 中会通过``THREAD\_POOL\_EXECUTOR`这个线程池执行任务，只有执行完当前任务，才会继续调用`scheduleNext`执行下一条任务，从此处可以看出，AsyncTask 默认是串行执行任务。在`r.run\(\)`中会调用 mWorker 的`call`方法，其中便会调用我们覆写的`doInBackground`方法，最后会执行`postResult\(result\)\`：

```text
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```

此处通过 Hander 发送一个 `MESSAGE_POST_RESULT` 类型的 Message，从子线程跳转到了 UI 线程进行处理：

```text
private static class InternalHandler extends Handler {
    public InternalHandler() {
        super(Looper.getMainLooper());
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
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

InternalHandler 中调用了 AsyncTask 的 `finish(Result)` 方法：

```text
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```

此处会判断当前 AsyncTask 是否被取消，取消调用 `onCancelled(result)`，否则调用 `onPostExecute(result)`，最后将 mStatus 设置为 FINISHED，防止二次使用。此处可见若要取消一个 AsyncTask，可以通过 `cancel(boolean)` 来取消，但应该保证 AsyncTask 尽可能快的取消，所以通常需要在 `doInBackground(Object[])` 中定期检查 `isCancelled()` 的返回值来结束异步任务。

