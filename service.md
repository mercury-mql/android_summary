# Service

***

**Service并不是一个独立的进程，除非通过android：process等指明，否则它运行在APP的进程中。**

**Service并不是一个线程，它运行在APP的主线程中**




## 生命周期

![](images/service_lifecycle.png)


Service分为两类：

### 1. Started Service

由其他组件调用startService而创建。一旦启动就在后台持续运行，即使启动它的组件被销毁也不受影响。**如果要终止，只能其自身调用stopSelf或者其他组件调用stopService。（而且无论调用了startService多少次，并不会造成嵌套，只要调用stopService或stopSelf一次，service就终止了）。**一旦被终止，系统将会回收它（onDestroy）。

onCreate - onStartCommand - onDestroy

多用于无需进行交互的场景。

**此类service必须实现onStartCommand**

### 2. Bound Service

由其他组件（作为客户端）通过调用bindService而创建（如果不存在则创建，但不会调用onStartCommand）。客户端可以得到onBind返回的IBinder对象，通过这个对象与service进行通信。客户端通过unbindService解除绑定。多个组件可以绑定在同一个service上。当所有的绑定者都解绑之后，系统自动销毁该service。

**此类service必须实现onBind**

onCreate - onBind - onUnbind - onDestroy

### 3. 两者混合

![](images/service_binding_tree_lifecycle.png)


When a service is unbound from all clients, the Android system destroys it (unless it was also started with onStartCommand()). As such, you don't have to manage the lifecycle of your service if it's purely a bound service—the Android system manages it for you based on whether it is bound to any clients.

However, if you choose to implement the onStartCommand() callback method, then you must explicitly stop the service, because the service is now considered to be started. In this case, the service runs until the service stops itself with stopSelf() or another component calls stopService(), regardless of whether it is bound to any clients.

Additionally, if your service is started and accepts binding, then when the system calls your onUnbind() method, you can optionally return true if you would like to receive a call to onRebind() the next time a client binds to the service. onRebind() returns void, but the client still receives the IBinder in its onServiceConnected() callback. Below, figure 1 illustrates the logic for this kind of lifecycle.




**注意：started service和bound service并不是完全分离的。可以绑定一个started service。**
例如：背景音乐service可以通过startService开启，当需要对某首乐曲进行精细的控制或者获取相关信息时，可以通过bindService将Activity与该Service绑定起来。

**注意：当started service被其他组件绑定后，调用stopService或stopSelf能否终止此service要根据bindService时指定的flags决定，如果指定了Context.BIND\_AUTO\_CREATE标记，就会终止此Service。换言之，只要Service（1）处于started状态（由startService启动，但没有调用stopService或stopSelf终止），或（2）存在一个或多个带有Context.BIND\_AUTO\_CREATE标记的连接，只要（1）和（2）满足任一个条件系统就会一直保持此service存在，如果两个条件都不具备，系统就会销毁掉此Service（onDestory），同时ServiceConnection的onServiceDisconnected回调被调用，表明是意外销毁**

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

- intent:   传递给startService的intent（实际上不是同一个对象）。
- flags:    额外数据。
  可能值有: 

    （1） 0：没有特别指明的信息

    （2） START\_FLAG\_REDELIVERY：当设置此标记时，表明intent是被re-delivery的（这是由于onStartCommand返回了START_REDELIVER_INTENT，而intent对应的service还没有执行到stopSelf就被终止了）

    （3） START_FLAG_RETRY：当设置此标志时，表明service是被异常终止后又重启的。

- startId:  代表当前service中的onStartCommand方法被调用的次数（可知onStartCommand第一次被调用时传入的startId值为1，在当前Service实例没被销毁的情况下，onStartCommand方法每被调用一次，传入的startId便会+1）。在当前Service实例被停止之后（例如调用了stopSelf()），Service若被再次启动，那启动的Service将是一个新的实例，startId将会从1开始重新计数。可用于stopSelfResult(int).


返回值：

当Service因内存等原因而被系统终止，然后由被系统重新启动时，onStartCommand的返回值将决定重新启动的模式

##### 4.2.1 START\_NOT\_STICKY

当Service的process被kill，不会被重建（哪怕当前这个intent还没有处理完），除非有started intent（被其他的startService发起，但还没有被处理）等待被处理。

如果onStartCommand返回START_NOT_STICKY，那即使service没有执行完，被杀掉后也不会被系统重启。如果这个service是用来为界面的Activity处理数据用的，那它不是必须一定要执行完的，大部分的情形是service所在的应用进程都已经被系统杀掉，这时没必要重启再次执行service。



此模式适用场景：service会被定时startService的。比如用alarm控制定时从服务器上取数据。

##### 4.2.2 START\_REDELIVER\_INTENT

当Service的process被kill，如果Service还没调用stopSelf，那么此Service将重建，onStartCommand将收到上次没有处理完的intent。


在此种模式下，onStartCommand的intent永远不会为null（因为第一次启动时，intent来自startService，不可能为null；重建时，只有存在未处理完的intent时才重建，其intent必然也非null）。


在重启时会重传被杀时未完成的Intent。比如该startService调用了4次，第1、2次的任务已经被service处理完（比如调用了stopSelf或stopService），第3、4次还未被处理时就被杀掉了，重启时会按顺序传入第3、4个Intent。重启后调用stopSelf的顺序要注意startId的顺序。因为第3、4次任务可能会被service 交给不同线程去执行，可能4先被执行完，如果4执行完后调用stopSelf(startId4)的话，那么3会被立即停止，即使它还没被执行完。所以stopSelf的顺序要严格按照收到onStartCommand中的startId来执行。


##### 4.2.3 START\_STICKY


当Service的process被kill，不会保存正在处理的intent，重建时传入onStartCommand中的intent为null。

被杀掉后系统会重启service，并且onStartCommand一定会被调用，如果在重启期间没有任何启动命令被传递到service，那么参数Intent将为null

因为START_STICKY类型默认传入的Intent为null，所以在使用时我们要仔细考虑。如果service需要使用Intent里的参数，那很有可能被重启时并没有调用者能传入这个参数。比如，该service是在某场景下才会被本应用的其他组件所调用启动，那么有可能整个应用都被杀掉了，重启该service时，只会经过Application的onCreate和该service的onCreate、onStartCommand，没有经过调用启动的上下文。或者是收到broadcast而触发该service，则重启期间可能不会收到broadcast。**只有当service是必须要完成的，并且不依赖于传入的Intent才需要把返回值设为START_STICKY。**

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

虽然可以有多个客户端绑定到service上，但系统仅在第一次绑定时调用onBind方法来获得IBinder对象。之后的绑定，系统将把同一个IBinder分发到其他的客户端。

参数：
<pre><code>
intent:	 传递给bindService的intent（实际上不是同一个对象）。注意，其中的extras在这里是看不到的。
</code></pre>

#### 4.5 onUnbind

<pre><code>
public boolean onUnbind (Intent intent)
</code></pre>

当**所有的**客户端都从service解绑时调用。此函数的默认实现仅仅返回false。

参数：
<pre><code>
intent:	 传递给bindService的intent（实际上不是同一个对象）。注意，其中的extras在这里是看不到的。
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

用startService启动的service会尽可能的保持运行（如果它所在的process转入后台，那么它的级别是服务process，其优先级低于前台process和可见process），只有当前台APP造成系统资源紧张时，系统才会kill它（并不多见）。如果发生了任何错误，此service会重启。

**注意：所使用的的intent要么指定完整的组件名（显式调用）；要么指定action，然后通过setPackage指定service的包名。**

### stopService
<pre><code>
Context.stopService (Intent service)
</code></pre>

停止服务。无论调用了多少次startService，一次stopService的调用就将终止此Service。

**注意：如果此Service在stopService时还存在使用BIND\_AUTO\_CREATE与客户端的绑定，它将不会被销毁（onDestroy）除非所有的绑定都解除。**

**注意：所使用的的intent要么指定完整的组件名（显式调用）；要么指定action，然后通过setPackage指定service的包名。**


### Service自己终止

#### stopSelf
<pre><code>
Service.stopSelf ()
</code></pre>

与context.stopService(intent)作用相同

#### stopSelf
<pre><code>
void stopSelf (int startId)
</code></pre>

与stopSelfResult作用相同，只不过没有返回值。

#### stopSelfResult
<pre><code>
Service.stopSelfResult (int startId)
</code></pre>

如果某个Service最后是以startId启动的，终止它。如果startId不是最后的，不终止。

这与Context.stopService(intent)作用相同，但可以避免一种情况，那就是“意外终止了尚未处理的start”
received them.

参数：startId是在onStartCommand (Intent intent, int flags, int startId)中收到的参数

返回值：当startId恰好是最后的startId时，终止service并返回true。否则返回false。


## Bound Service

### bindService

<pre><code>
Context.bindService (Intent intent, ServiceConnection conn, int flags)
</code></pre>

绑定服务。必要时创建服务。

**注意：此方法不能再BrodcastReceiver的onReceive中调用，因为其context为ReceiverRestrictedContext（禁用了bindService和registerReceiver），根本原因广播本身生命周期很短，bind的话没有意义。如果必须要在receiver中创建service，则应使用startService（启动的service需要自己调用stopSelf）

参数：

intent： **要么指定完整的组件名（显式调用）；要么指定action，然后通过setPackage指定service的包名。**

conn： ServiceConnection对象。不可为null。

flags：可以为以下值：

- 0：不新建服务。**如果绑定到一个以startService开启的Service上，绑定后，此Service又被调用了stopService，由于标志被设为0（没有设置BIND\_AUTO\_CREATE），ServiceConnection的onServiceDisconnected不会被调用，随后这个Service被销毁（onDestory）。**

- BIND\_AUTO\_CREATE：只要绑定存在，就自动的创建该service。**如果绑定到一个以startService开启的Service上，绑定后，此Service又被调用了stopService，由于此标志的存在，这个Service不会被销毁（onDestory）。当调用unbindService解绑后，此Service才销毁，但ServiceConnection的onServiceDisconnected不会被调用，因为这是主动解绑**

- BIND\_DEBUG\_UNBIND：会造成内存泄漏，仅用于调试。

- BIND\_NOT\_FOREGROUND：禁止由于绑定而将此Service的“调度优先级”升为前台。**注意：它的”内存优先级”将不低于客户端的"内存优先级"（也就是说只要客户端不能被kill，那么此service也不能被kill，就如在《process与thread》中的描述）。在CPU调度此service时，仍然把它作为后台。这仅影响到客户端是前台进程，而Service是后台进程的场景。

- BIND\_ABOVE\_CLIENT：指明被绑定的service比客户端的内存优先级还要高（即使客户端被kill，此service也不能被kill）。

- BIND\_ALLOW\_OOM\_MANAGEMENT：允许service所在的process完成正常的内存管理。在service运行时，允许系统因内存不足等突发性的情况而临时的杀死此service，并且此service如果运行了较长的时间，将被认为是更合适的被kill的对象。

- BIND\_WAIVE\_PRIORITY：设置后不影响service所在process的调度优先级和内存优先级。允许service所在process向普通的运行在后台的APP的process一样通过后台LRU列表管理。

**0 和BIND\_AUTO\_CREATE比较常用**

返回值：成功绑定返回true

**注意：此方法不会导致onStartCommand被调用**

### unbindService

<pre><code>
Context.unbindService (ServiceConnection conn)
</code></pre>

解除绑定。

### ServiceConnection的说明

#### onServiceConnected
<pre><code>
public abstract void onServiceConnected (ComponentName name, IBinder service)
</code></pre>

绑定建立时回调。

参数：
name：    被连接的service的名称。
service： IBinder对象。实际上就是实现Service时，onBind返回的对象。

#### onServiceDisconnected
<pre><code>
public abstract void onServiceDisconnected (ComponentName name)
</code></pre>

当绑定意外断开后回调。比如service所在的process崩溃或被kill。

参数：

name： 连接意外断开的service的名称。

### 绑定Service的实现

当创建Bound Service时，必须要实现onBind函数，同时返回IBinder对象。创建IBinder对象有三种方式：
- 继承Binder类
- 使用Messenger
- 使用AIDL

#### 1. 继承Binder类

适用场景：service是**APP私有的，运行在与客户端相同的process中**。

这是当service仅作为你的APP的一个幕后工作者时的绝佳方式。只有当service被其他APP使用或进行跨进程使用时，才应该考虑其他方式。

步骤：

（1）创建类（类名为LocalService），继承自Service。在LocalService类中，创建一个内部类（类名为LocalBinder），这个类继承自Binder类，它需要提供如下功能：
- 提供public方法，返回此Service的实例(比如getService()),此方法一般在ServiceConnection中的onServiceConnected中调用，从而使客户端可以访问到此Service。**此功能必须提供**
- 提供public方法，返回此Service内部持有的某个类的实例
- 提供其他可供客户端访问的public方法

（2）在LocalService中，实现onBind方法，在该方法中，返回LocalBinder的实例。（此实例将作为ServiceConnection中onServiceConnected的一个参数）

（3）在客户端中，定义ServiceConnection对象，在其中的onServiceConnected方法中，通过强转将IBinder对象转化为LocalBinder对象，调用其中的getService()，从而获取LocalService的实例（这才是**最终目的**，此实例多保存为成员变量）。

（4）在客户端中，调用bindService绑定LocalService

（5）通过调用保存的LocalService实例的方法适时进行交互。**注意，如果交互会导致主线程阻塞，应在工作线程中进行**

**注意：service必须与客户端在同一个process中的原因就在于，客户端可以通过强转的方式将IBinder对象转化为MyBinder，并调用它的public方法获取service的实例。**

<pre><code>

public class LocalService extends Service {

    // 创建LocalBinder的实例，将作为onBind的返回值
    private final IBinder mBinder = new LocalBinder();

    // 随机数发生器
    private final Random mGenerator = new Random();

    // 内部类，继承自Binder
    public class LocalBinder extends Binder {

		// 返回此Service的实例，一般在ServiceConnection的onServiceConnected中被调用
        public LocalService getService() {
            
            return LocalService.this;
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }

    // service的方法。当客户端得到此service的实例后，可以调用
    public int getRandomNumber() {
      return mGenerator.nextInt(100);
    }
}

</code></pre>

客户端

<pre><code>

public class BindingActivity extends Activity {

	// LocalService的实例
    LocalService mService;

    boolean mBound = false;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
    }

    @Override
    protected void onStart() {
        super.onStart();
        
		// Bind to LocalService
        Intent intent = new Intent(this, LocalService.class);
        
		// 绑定服务
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }
    @Override
    protected void onStop() {
        super.onStop();

        // 解绑
        if (mBound) {
            unbindService(mConnection);
            mBound = false;
        }
    }
    
    public void onButtonClick(View v) {
        if (mBound) {
            // 交互。注意，如果交互会导致主线程阻塞，应在工作线程中进行
            int num = mService.getRandomNumber();
            Toast.makeText(this, "number: " + num, Toast.LENGTH_SHORT).show();
        }
    }
    
    //定义ServiceConnection对象
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName className,
                IBinder service) {
            
            // 强转为LocalBinder对象
            LocalBinder binder = (LocalBinder) service;

            // 获取LocalService实例，保存为成员变量
            mService = binder.getService();
            mBound = true;
        }
        @Override
        public void onServiceDisconnected(ComponentName arg0) {
            mBound = false;
        }
    };
}

</code></pre>
The above sample shows how the client binds to the service using an implementation of ServiceConnection and the onServiceConnected() callback. The next section provides more information about this process of binding to the service.

Note: The example above doesn't explicitly unbind from the service, but all clients should unbind at an appropriate time (such as when the activity pauses).

For more sample code, see the LocalService.java class and the LocalServiceActivities.java class in ApiDemos.

#### 2. 使用Messenger

If you need your interface to work across different processes, you can create an interface for the service with a Messenger. In this manner, the service defines a Handler that responds to different types of Message objects. This Handler is the basis for a Messenger that can then share an IBinder with the client, allowing the client to send commands to the service using Message objects. Additionally, the client can define a Messenger of its own so the service can send messages back.
This is the simplest way to perform interprocess communication (IPC), because the Messenger queues all requests into a single thread so that you don't have to design your service to be thread-safe.

#### 3. 使用AIDL

AIDL (Android Interface Definition Language) performs all the work to decompose objects into primitives that the operating system can understand and marshall them across processes to perform IPC. The previous technique, using a Messenger, is actually based on AIDL as its underlying structure. As mentioned above, the Messenger creates a queue of all the client requests in a single thread, so the service receives requests one at a time. If, however, you want your service to handle multiple requests simultaneously, then you can use AIDL directly. In this case, your service must be capable of multi-threading and be built thread-safe.
To use AIDL directly, you must create an .aidl file that defines the programming interface. The Android SDK tools use this file to generate an abstract class that implements the interface and handles IPC, which you can then extend within your service.

Note: Most applications should not use AIDL to create a bound service, because it may require multithreading capabilities and can result in a more complicated implementation. As such, AIDL is not suitable for most applications and this document does not discuss how to use it for your service. If you're certain that you need to use AIDL directly, see the AIDL document.

## 前台Service

## AIDL

## 更便捷的方式——IntentService

[http://android.xsoftlab.net/guide/components/services.html](http://android.xsoftlab.net/guide/components/services.html)

[http://android.xsoftlab.net/reference/android/app/Service.html](http://android.xsoftlab.net/reference/android/app/Service.html)

## 疑问

1. startService的intent与onStartCommand的intent是一个对象吗？
不是。
2. stopSelfResult中的startId是或不是最后的startId，有什么不同？
是最后的，终止service，返回true。不是最后的，不终止，返回false。