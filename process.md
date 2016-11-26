# API GUIDE学习——进程与线程

*本篇有一部分内容取自Api Guide中的《进程与线程》*

***

一个APP中含有多个组件（component），如果其中某个组件要开始运行，有两种情况：

- 该APP中没有其他组件处于运行状态，那么系统将会为该APP创建一个Linux process，这个process仅有一个thread运行其中（主线程）。默认情况下，所有的组件都运行在这个主线程中。

- 该APP中已经有其他组件处于运行状态（也就是说，已经有一个process存在了），那么这个组件将会在这个process中运行（并运行在同样的主thread中）。

默认情况下，APP中的所有组件都运行在主线程中。当然，可以通过其他方式使APP按照多process、多thread的方式运行。


## Process

默认情况下，所有component运行在同一个process中，这在绝大多数情况下都是合适的。

当确实有多process的需求时，可以在xml文件中为<activity\>, <service\>, <receiver\>, <provider\>等组件指定android:process属性，该属性指明了该组件要运行在哪一个process中。

<application\>标签同样支持android:process属性，它为各个组件指定了一个默认值。

回收资源时，process被kill的顺序与它的重要性级别（或者说优先级）相关，一个process的重要性由两部分决定：

- 运行在process内部的component类型
- component的当前状态

重要性级别有高到低分为以下五类：

### 1. 前台Process

是当前用户的操作所必需的。有以下几种情况：

- 包含Activity，该Activity正在与用户进行交互（Activity的onResume被调用）

- 包含Service，该Service与一个正在与用户交互的Activity绑定

- 包含Service，该Service被设为前台的（Service被调用了startFroeground）

- 包含Service，该Service运行在某个生命周期的方法中（如onCreate、onStart、onDestroy）

- 包含BroadcastReceiver，该receiver运行在onReceive方法中

通常情况下，在任何时刻下，只有一部分前台process存在。

只有当系统内存处于极端紧张的情况下，已经无法保证所有的前台Process都继续运行的情况下，一部分前台Process才会被终止。

### 2. 可见Process

不含有任何运行在前台的组件，但是能够影响到用户在屏幕上看到的内容。有以下几种情况：

- 包含Activity，它能够被看见但不处于前台（Activity的onPause被调用）。比较常见的情况是Activity启动了一个Dialog，Activity被部分遮挡，此时Activity就是可见但非前台。

- 包含Service，该Service与一个正处于可见状态的Activity绑定

可见Process也被认为是极端重要的，当内存严重不足时，为了保证所有的前台Process都能运行，才会考虑kill可见的Process。 


### 3. 服务Process

包含Service（通过startService启动的），同时不再上述两种情况之内的Process。

虽然服务Process不会被用户看到，但它在幕后做的工作是用户关心的（比如播放音乐、下载数据等），所以除非为了保证前台Process和可见Process都能够运行，否则系统不会kill掉它。

### 4. 后台Process

包含Activity，而该Activity当前是不可见的（Activity的onStop被调用）。

后台Process不会影响到用户体验，所以为保证前台Process、可见Process、服务Process能够顺利运行，系统可能随时kill掉它。

通常，系统内会有多个后台Process，它们被组织在一个LRU列表中（least recently used），这样就保证了用户包含最近用户看到的Activity的后台Process会被最后kill。如果Activity的生命周期方法被正确的实现，并且它的状态被保存，那么当这个后台Process被kill时不会对用户体验造成影响。

### 5. 空Process

不包含任何“活跃的”组件，系统保留这类Process的目的在于为了提高APP的启动时间而保持缓存。系统会适时kill掉它。

### 级别确定

系统会根据运行在某一Process中的“活动的”组件的最高优先级来确定该Process的级别。<br/>例如：一个Process包含一个可见的Activity和一个非绑定的Service，那么该Process被确定为可见Process。

此外，Process之间的依赖关系也会导致Process级别的提升。一个为其他Process（暂称为B）提供支持的Process（暂称为A），那么A的级别不会低于B。

这里有两个例子：

- Process A中运行着一个Content Provider，它为运行在Process B中的一个客户端提供支持，那么A至少被认为与B一样重要。

- Process A中运行着一个Service，该Service与运行在Process B中的某一个组件绑定，那么A也至少被认为与B一样重要。

因为服务Process的级别高于后台Process，所以如果一个Activity需要执行耗时操作，它不应该启动一个工作Thread，而应该启动一个Service（可以在Service中启thread，或直接使用IntentService），尤其是当这个操作可能比Activity存在的时间长的情况下。

例如：一个Activity需要向某个网站上传图片，它可以启动一个Service执行操作，那么即使用户关闭了这个Activity，Service依然可以在后台执行上传。

使用Service保证了无论该Activity发生了什么，其执行的操作至少是“服务Process”的级别。这也是在BroadcastReceiver的onReceive中启动Service执行动作（而非直接在onReceive中执行）的原因。

## Thread

当一个APP启动时，其Process中仅包含一个Thread（被称为主线程，也被称为UI线程），它负责向UI组件分发事件，包括各类交互、绘制等。

系统所有的组件都运行在同一个Thread中（主线程），而且针对每个组件的系统调用都是由这个线程分发的，所以各类回调（包括生命周期相关的、onKeyDown等用户操作相关的）都在这个Thread中运行。

例如：当用户点击了屏幕上的一个Button，你的APP的UI线程将会把Touch Event分发给widget，进而会更新它的按压状态并将invalidate请求放入event queue，UI线程会从event queue中取出这一请求，并通知widget重绘。

可以想见，把所有工作都放在UI线程不是什么good idea，轻则卡顿，重则ANR。

另外，Android不允许在非UI线程更新UI组件，因为这不是“线程安全”的。

有两条原则必须遵循：

- 不能阻塞UI线程

- 在其他线程不能访问UI组件

### 工作Thread

鉴于上面的讨论，如果执行耗时任务，启工作Thread势在必行。

同时为了能够在工作Thread中访问UI组件，系统提供了如下函数：

<pre><code>
Activity.runOnUiThread(Runnable r)

View.post(Runnable r)

View.postDelayed(Runnable r, long delay)

</code></pre>

例如：
<pre><code>
public void onClick(View v) {
    new Thread(new Runnable() {
        public void run() {
            final Bitmap bitmap =
                    loadImageFromNetwork("http://example.com/image.png");
            mImageView.post(new Runnable() {
                public void run() {
                    mImageView.setImageBitmap(bitmap);
                }
            });
        }
    }).start();
}
</code></pre>

这样就实现了在工作Thread中访问网络，同时由能够访问到UI组件的目的。

但是随着业务量的增大，此类代码将变得不容易维护，处理更复杂的情况可以借助Handler来改善。此外采用AsyncTask也是不错的方式。


## AsyncTask


AsyncTask allows you to perform asynchronous work on your user interface. It performs the blocking operations in a worker thread and then publishes the results on the UI thread, without requiring you to handle threads and/or handlers yourself.

To use it, you must subclass AsyncTask and implement the doInBackground() callback method, which runs in a pool of background threads. To update your UI, you should implement onPostExecute(), which delivers the result from doInBackground() and runs in the UI thread, so you can safely update your UI. You can then run the task by calling execute() from the UI thread.

For example, you can implement the previous example using AsyncTask this way:

public void onClick(View v) {
    new DownloadImageTask().execute("http://example.com/image.png");
}
private class DownloadImageTask extends AsyncTask<String, Void, Bitmap> {
    /** The system calls this to perform work in a worker thread and
      * delivers it the parameters given to AsyncTask.execute() */
    protected Bitmap doInBackground(String... urls) {
        return loadImageFromNetwork(urls[0]);
    }
    /** The system calls this to perform work in the UI thread and delivers
      * the result from doInBackground() */
    protected void onPostExecute(Bitmap result) {
        mImageView.setImageBitmap(result);
    }
}
Now the UI is safe and the code is simpler, because it separates the work into the part that should be done on a worker thread and the part that should be done on the UI thread.

You should read the AsyncTask reference for a full understanding on how to use this class, but here is a quick overview of how it works:

You can specify the type of the parameters, the progress values, and the final value of the task, using generics
The method doInBackground() executes automatically on a worker thread
onPreExecute(), onPostExecute(), and onProgressUpdate() are all invoked on the UI thread
The value returned by doInBackground() is sent to onPostExecute()
You can call publishProgress() at anytime in doInBackground() to execute onProgressUpdate() on the UI thread
You can cancel the task at any time, from any thread
Caution: Another problem you might encounter when using a worker thread is unexpected restarts in your activity due to a runtime configuration change (such as when the user changes the screen orientation), which may destroy your worker thread. To see how you can persist your task during one of these restarts and how to properly cancel the task when the activity is destroyed, see the source code for the Shelves sample application.



类文档

Class Overview
AsyncTask enables proper and easy use of the UI thread. This class allows to perform background operations and publish results on the UI thread without having to manipulate threads and/or handlers.

AsyncTask is designed to be a helper class around Thread and Handler and does not constitute a generic threading framework. AsyncTasks should ideally be used for short operations (a few seconds at the most.) If you need to keep threads running for long periods of time, it is highly recommended you use the various APIs provided by the java.util.concurrent package such as Executor, ThreadPoolExecutor and FutureTask.

An asynchronous task is defined by a computation that runs on a background thread and whose result is published on the UI thread. An asynchronous task is defined by 3 generic types, called Params, Progress and Result, and 4 steps, called onPreExecute, doInBackground, onProgressUpdate and onPostExecute.

Developer Guides

For more information about using tasks and threads, read the Processes and Threads developer guide.

Usage
AsyncTask must be subclassed to be used. The subclass will override at least one method (doInBackground(Params...)), and most often will override a second one (onPostExecute(Result).)

Here is an example of subclassing:

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
 
Once created, a task is executed very simply:

 new DownloadFilesTask().execute(url1, url2, url3);
 
AsyncTask's generic types
The three types used by an asynchronous task are the following:

Params, the type of the parameters sent to the task upon execution.
Progress, the type of the progress units published during the background computation.
Result, the type of the result of the background computation.
Not all types are always used by an asynchronous task. To mark a type as unused, simply use the type Void:

 private class MyTask extends AsyncTask<Void, Void, Void> { ... }
 
The 4 steps
When an asynchronous task is executed, the task goes through 4 steps:

onPreExecute(), invoked on the UI thread before the task is executed. This step is normally used to setup the task, for instance by showing a progress bar in the user interface.
doInBackground(Params...), invoked on the background thread immediately after onPreExecute() finishes executing. This step is used to perform background computation that can take a long time. The parameters of the asynchronous task are passed to this step. The result of the computation must be returned by this step and will be passed back to the last step. This step can also use publishProgress(Progress...) to publish one or more units of progress. These values are published on the UI thread, in the onProgressUpdate(Progress...) step.
onProgressUpdate(Progress...), invoked on the UI thread after a call to publishProgress(Progress...). The timing of the execution is undefined. This method is used to display any form of progress in the user interface while the background computation is still executing. For instance, it can be used to animate a progress bar or show logs in a text field.
onPostExecute(Result), invoked on the UI thread after the background computation finishes. The result of the background computation is passed to this step as a parameter.
Cancelling a task
A task can be cancelled at any time by invoking cancel(boolean). Invoking this method will cause subsequent calls to isCancelled() to return true. After invoking this method, onCancelled(Object), instead of onPostExecute(Object) will be invoked after doInBackground(Object[]) returns. To ensure that a task is cancelled as quickly as possible, you should always check the return value of isCancelled() periodically from doInBackground(Object[]), if possible (inside a loop for instance.)

Threading rules
There are a few threading rules that must be followed for this class to work properly:

The AsyncTask class must be loaded on the UI thread. This is done automatically as of JELLY_BEAN.
The task instance must be created on the UI thread.
execute(Params...) must be invoked on the UI thread.
Do not call onPreExecute(), onPostExecute(Result), doInBackground(Params...), onProgressUpdate(Progress...) manually.
The task can be executed only once (an exception will be thrown if a second execution is attempted.)
Memory observability
AsyncTask guarantees that all callback calls are synchronized in such a way that the following operations are safe without explicit synchronizations.

Set member fields in the constructor or onPreExecute(), and refer to them in doInBackground(Params...).
Set member fields in doInBackground(Params...), and refer to them in onProgressUpdate(Progress...) and onPostExecute(Result).
Order of execution
When first introduced, AsyncTasks were executed serially on a single background thread. Starting with DONUT, this was changed to a pool of threads allowing multiple tasks to operate in parallel. Starting with HONEYCOMB, tasks are executed on a single thread to avoid common application errors caused by parallel execution.

If you truly want parallel execution, you can invoke executeOnExecutor(java.util.concurrent.Executor, Object[]) with THREAD_POOL_EXECUTOR.




## 线程安全的方法

某些情况下,我们实现的方法可能会被多个线程调用，这就涉及到了线程安全的问题。


This is primarily true for methods that can be called remotely—such as methods in a bound service. When a call on a method implemented in an IBinder originates in the same process in which the IBinder is running, the method is executed in the caller's thread. However, when the call originates in another process, the method is executed in a thread chosen from a pool of threads that the system maintains in the same process as the IBinder (it's not executed in the UI thread of the process). For example, whereas a service's onBind() method would be called from the UI thread of the service's process, methods implemented in the object that onBind() returns (for example, a subclass that implements RPC methods) would be called from threads in the pool. Because a service can have more than one client, more than one pool thread can engage the same IBinder method at the same time. IBinder methods must, therefore, be implemented to be thread-safe.

Similarly, a content provider can receive data requests that originate in other processes. Although the ContentResolver and ContentProvider classes hide the details of how the interprocess communication is managed, ContentProvider methods that respond to those requests—the methods query(), insert(), delete(), update(), and getType()—are called from a pool of threads in the content provider's process, not the UI thread for the process. Because these methods might be called from any number of threads at the same time, they too must be implemented to be thread-safe.

## Interprocess Communication
Android offers a mechanism for interprocess communication (IPC) using remote procedure calls (RPCs), in which a method is called by an activity or other application component, but executed remotely (in another process), with any result returned back to the caller. This entails decomposing a method call and its data to a level the operating system can understand, transmitting it from the local process and address space to the remote process and address space, then reassembling and reenacting the call there. Return values are then transmitted in the opposite direction. Android provides all the code to perform these IPC transactions, so you can focus on defining and implementing the RPC programming interface.

To perform IPC, your application must bind to a service, using bindService(). For more information, see the Services developer guide.




Processes and Application Life Cycle
In most cases, every Android application runs in its own Linux process. This process is created for the application when some of its code needs to be run, and will remain running until it is no longer needed and the system needs to reclaim its memory for use by other applications.

An unusual and fundamental feature of Android is that an application process's lifetime is not directly controlled by the application itself. Instead, it is determined by the system through a combination of the parts of the application that the system knows are running, how important these things are to the user, and how much overall memory is available in the system.

It is important that application developers understand how different application components (in particular Activity, Service, and BroadcastReceiver) impact the lifetime of the application's process. Not using these components correctly can result in the system killing the application's process while it is doing important work.

A common example of a process life-cycle bug is a BroadcastReceiver that starts a thread when it receives an Intent in its BroadcastReceiver.onReceive() method, and then returns from the function. Once it returns, the system considers the BroadcastReceiver to be no longer active, and thus, its hosting process no longer needed (unless other application components are active in it). So, the system may kill the process at any time to reclaim memory, and in doing so, it terminates the spawned thread running in the process. The solution to this problem is to start a Service from the BroadcastReceiver, so the system knows that there is still active work being done in the process.

To determine which processes should be killed when low on memory, Android places each process into an "importance hierarchy" based on the components running in them and the state of those components. These process types are (in order of importance):

A foreground process is one that is required for what the user is currently doing. Various application components can cause its containing process to be considered foreground in different ways. A process is considered to be in the foreground if any of the following conditions hold:
It is running an Activity at the top of the screen that the user is interacting with (its onResume() method has been called).
It has a BroadcastReceiver that is currently running (its BroadcastReceiver.onReceive() method is executing).
It has a Service that is currently executing code in one of its callbacks (Service.onCreate(), Service.onStart(), or Service.onDestroy()).
There will only ever be a few such processes in the system, and these will only be killed as a last resort if memory is so low that not even these processes can continue to run. Generally, at this point, the device has reached a memory paging state, so this action is required in order to keep the user interface responsive.

A visible process is one holding an Activity that is visible to the user on-screen but not in the foreground (its onPause() method has been called). This may occur, for example, if the foreground Activity is displayed as a dialog that allows the previous Activity to be seen behind it. Such a process is considered extremely important and will not be killed unless doing so is required to keep all foreground processes running.
A service process is one holding a Service that has been started with the startService() method. Though these processes are not directly visible to the user, they are generally doing things that the user cares about (such as background mp3 playback or background network data upload or download), so the system will always keep such processes running unless there is not enough memory to retain all foreground and visible process.
A background process is one holding an Activity that is not currently visible to the user (its onStop() method has been called). These processes have no direct impact on the user experience. Provided they implement their Activity life-cycle correctly (see Activity for more details), the system can kill such processes at any time to reclaim memory for one of the three previous processes types. Usually there are many of these processes running, so they are kept in an LRU list to ensure the process that was most recently seen by the user is the last to be killed when running low on memory.
An empty process is one that doesn't hold any active application components. The only reason to keep such a process around is as a cache to improve startup time the next time a component of its application needs to run. As such, the system will often kill these processes in order to balance overall system resources between these empty cached processes and the underlying kernel caches.
When deciding how to classify a process, the system will base its decision on the most important level found among all the components currently active in the process. See the Activity, Service, and BroadcastReceiver documentation for more detail on how each of these components contribute to the overall life-cycle of a process. The documentation for each of these classes describes in more detail how they impact the overall life-cycle of their application.

A process's priority may also be increased based on other dependencies a process has to it. For example, if process A has bound to a Service with the Context.BIND_AUTO_CREATE flag or is using a ContentProvider in process B, then process B's classification will always be at least as important as process A's.