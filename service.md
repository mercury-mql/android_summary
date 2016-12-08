# Service

***

**Service并不是一个独立的进程，除非通过android：process等指明，否则它运行在APP的进程中。**

**Service并不是一个线程，它运行在APP的主线程中**




## 生命周期

![](images/service_lifecycle.png)


Service分为两类：

### 1. A started service

由其他组件调用startService而创建。一旦启动就在后台持续运行，即使启动它的组件被销毁也不受影响。**如果要终止，只能其自身调用stopSelf或者其他组件调用stopService。（而且无论调用了startService多少次，并不会造成嵌套，只要调用stopService或stopSelf一次，service就终止了）。**一旦被终止，系统将会回收它（onDestroy）。

onCreate - onStartCommand - onDestroy

多用于无需进行交互的场景。

**此类service必须实现onStartCommand**

### 2. A bound service

由其他组件（作为客户端）通过调用bindService而创建（如果不存在则创建，但不会调用onStartCommand）。客户端可以得到onBind返回的IBinder对象，通过这个对象与service进行通信。客户端通过unbindService解除绑定。多个组件可以绑定在同一个service上。当所有的绑定者都解绑之后，系统自动销毁该service。

**此类service必须实现onBind**

onCreate - onBind - onUnbind - onDestroy

### 3. 两者混合

**注意：started service和bound service并不是完全分离的。可以绑定一个started service。**
例如：背景音乐service可以通过startService开启，当需要对某首乐曲进行精细的控制或者获取相关信息时，可以通过bindService将Activity与该Service绑定起来。

**注意：当started service被其他组件绑定后，仅调用stopService或stopSelf并不能终止此service，需要等到所有绑定到它上面的客户端都解绑才行。**

**只要service是started的，或存在一个或多个带有Context.BIND_AUTO_CREATE标记的连接，系统就会一直保持此service存在。**

### 4. 生命周期方法

#### 4.1 onCreate
<pre><code>
public void onCreate ()
</code></pre>

创建。

#### 4.2 onStartCommand
<pre><code>
public int onStartCommand (Intent intent, int flags, int startId)
</code></pre>

仅在每次其他组件调用startService时被回调。

**注意：此函数在APP的主线程中执行。如果需要执行长时间操作、网络调用、I/O操作，应开启Thread或使用AsyncTask。**

参数：
<pre><code>
intent:   传递给startService的intent。
flags:    额外数据。可能值有: 0、START_FLAG_REDELIVERY、START_FLAG_RETRY.
startId:  start的标识。可用于stopSelfResult(int).
</code></pre>

返回值：

当Service因内存等原因而被系统终止，然后由被系统重新启动时，onStartCommand的返回值将决定重新启动的模式

##### 4.2.1 START\_NOT\_STICKY

当Service的process被kill，不会被重建，除非有started intent（被其他的startService发起，但还没有被处理）等待被处理。

此模式适用场景：service会被定时startService的。比如用alarm控制定时从服务器上取数据。

##### 4.2.2 START\_REDELIVER\_INTENT

当Service的process被kill，如果Service还没调用stopSelf，那么此Service将重建，onStartCommand将收到上次没有处理完的intent。

在此种模式下，onStartCommand的intent永远不会为null（因为第一次启动时，intent来自startService，不可能为null；重建时，只有存在未处理完的intent时才重建，其intent必然也非null）。


##### 4.2.3 START\_STICKY


当Service的process被kill，不会保存正在处理的intent，重建时传入onStartCommand中的intent为null。

这种模式适用于需要**显式**startService或stopService的场景，比如背景音乐。


#### 4.3 onDestroy
<pre><code>
public void onDestroy ()
</code></pre>

销毁。需要进行资源清理工作（销毁thread、注销receiver等）

#### 4.4 onBind
<pre><code>
public abstract IBinder onBind (Intent intent)
</code></pre>

返回与service进行通信的通道。如果不希望service被客户端绑定（纯started service），可以返回null。

参数：
<pre><code>
intent:	 传递给bindService的intent。注意，其中的extras在这里是看不到的。
</code></pre>

#### 4.5 onUnbind

<pre><code>
public boolean onUnbind (Intent intent)
</code></pre>

当**所有的**客户端都从service解绑时调用。此函数的默认实现仅仅返回false。

参数：
<pre><code>
intent:	 传递给bindService的intent。注意，其中的extras在这里是看不到的。
</code></pre>

返回值：**如果希望在所有的客户端都解绑后，当新的客户端再次绑定时能够回调到onRebind函数，那么应返回true**，默认实现返回false。

#### 4.6 onRebind
<pre><code>
public void onRebind (Intent intent)
</code></pre>

被回调的条件（需同时满足）：

- 所有的客户端都解绑后，有一个新的客户端希望绑定
- 在onUnbind中返回了true

参数：
<pre><code>
intent:	 传递给bindService的intent。注意，其中的extras在这里是看不到的。
</code></pre>

## Started Service

### startService
<pre><code>
Context.startService (Intent service)
</code></pre>

启动服务。如果该Service已经处于运行状态，则保持运行。如果该Service不存在，创建并运行之。

每次调用此函数，必然会导致Service.onStartCommand被调用，onStartCommand中的intent就是这里的参数。（这相当于提供了一个在无需绑定的情况下提交作业的便捷方式）

startService将改变service的存活时间（如果此service是由bindService启动的），即使没有任何客户端绑定在上面，只要还没有调用stopService，那么这个service就一直处于运行状态。

**注意：startService不会被嵌套，无论调用了多少次startService，一次stopService的调用就将终止此Service。**



The system attempts to keep running services around as much as possible. The only time they should be stopped is if the current foreground application is using so many resources that the service needs to be killed. If any errors happen in the service's process, it will automatically be restarted.

This function will throw SecurityException if you do not have permission to start the given service.

Parameters
service	Identifies the service to be started. The Intent must be either fully explicit (supplying a component name) or specify a specific package name it is targetted to. Additional values may be included in the Intent extras to supply arguments along with this specific start call.
Returns
If the service is being started or is already running, the ComponentName of the actual service that was started is returned; else if the service does not exist null is returned.

### stopService

public abstract boolean stopService (Intent service)

Added in API level 1
Request that a given application service be stopped. If the service is not running, nothing happens. Otherwise it is stopped. Note that calls to startService() are not counted -- this stops the service no matter how many times it was started.

Note that if a stopped service still has ServiceConnection objects bound to it with the BIND_AUTO_CREATE set, it will not be destroyed until all of these bindings are removed. See the Service documentation for more details on a service's lifecycle.

This function will throw SecurityException if you do not have permission to stop the given service.

Parameters
service	Description of the service to be stopped. The Intent must be either fully explicit (supplying a component name) or specify a specific package name it is targetted to.
Returns
If there is a service matching the given Intent that is already running, then it is stopped and true is returned; else false is returned.

### stopSelf

public final void stopSelf ()

Added in API level 1
Stop the service, if it was previously started. This is the same as calling stopService(Intent) for this particular service.

See Also
stopSelfResult(int)
public final void stopSelf (int startId)

Added in API level 1
Old version of stopSelfResult(int) that doesn't return a result.

See Also
stopSelfResult(int)
public final boolean stopSelfResult (int startId)

Added in API level 1
Stop the service if the most recent time it was started was startId. This is the same as calling stopService(Intent) for this particular service but allows you to safely avoid stopping if there is a start request from a client that you haven't yet seen in onStart(Intent, int).

Be careful about ordering of your calls to this function.. If you call this function with the most-recently received ID before you have called it for previously received IDs, the service will be immediately stopped anyway. If you may end up processing IDs out of order (such as by dispatching them on separate threads), then you are responsible for stopping them in the same order you received them.

Parameters
startId	The most recent start identifier received in onStart(Intent, int).
Returns
Returns true if the startId matches the last start request and the service will be stopped, else false.

## Bound Service

### bindService

public abstract boolean bindService (Intent service, ServiceConnection conn, int flags)

Added in API level 1
Connect to an application service, creating it if needed. This defines a dependency between your application and the service. The given conn will receive the service object when it is created and be told if it dies and restarts. The service will be considered required by the system only for as long as the calling context exists. For example, if this Context is an Activity that is stopped, the service will not be required to continue running until the Activity is resumed.

This function will throw SecurityException if you do not have permission to bind to the given service.

Note: this method can not be called from a BroadcastReceiver component. A pattern you can use to communicate from a BroadcastReceiver to a Service is to call startService(Intent) with the arguments containing the command to be sent, with the service calling its stopSelf(int) method when done executing that command. See the API demo App/Service/Service Start Arguments Controller for an illustration of this. It is okay, however, to use this method from a BroadcastReceiver that has been registered with registerReceiver(BroadcastReceiver, IntentFilter), since the lifetime of this BroadcastReceiver is tied to another object (the one that registered it).

Parameters
service	Identifies the service to connect to. The Intent may specify either an explicit component name, or a logical description (action, category, etc) to match an IntentFilter published by a service.
conn	Receives information as the service is started and stopped. This must be a valid ServiceConnection object; it must not be null.
flags	Operation options for the binding. May be 0, BIND_AUTO_CREATE, BIND_DEBUG_UNBIND, BIND_NOT_FOREGROUND, BIND_ABOVE_CLIENT, BIND_ALLOW_OOM_MANAGEMENT, or BIND_WAIVE_PRIORITY.
Returns
If you have successfully bound to the service, true is returned; false is returned if the connection is not made so you will not receive the service object.

### unbindService

public abstract void unbindService (ServiceConnection conn)

Added in API level 1
Disconnect from an application service. You will no longer receive calls as the service is restarted, and the service is now allowed to stop at any time.

Parameters
conn	The connection interface previously supplied to bindService(). This parameter must not be null.

## 前台Service

## AIDL

## 更便捷的方式——IntentService

[http://android.xsoftlab.net/guide/components/services.html](http://android.xsoftlab.net/guide/components/services.html)

[http://android.xsoftlab.net/reference/android/app/Service.html](http://android.xsoftlab.net/reference/android/app/Service.html)

## 疑问

1. startService的intent与onStartCommand的intent是一个对象吗