# Broadcast Receiver

***

## 应用场景

- 同一APP内部的同一组件内的消息通信（单个或多个线程之间）

- 同一APP内部的不同组件间的消息通信（单个进程）

- 同一APP内部的具有不同进程的多个组件间的消息通信（多个进程）

- 不同APP组件间的消息通信（多个进程）

- Android系统与APP之间的消息通信（系统广播）

## 分类

### 1. 普通广播

<pre>
public abstract void sendBroadcast (Intent intent, String receiverPermission)

public abstract void sendBroadcast (Intent intent)
</pre>
所有receiver都是无序的，各receiver往往同时运行。相对来说更为高效，但各receiver无法终止广播或使用其他广播执行的结果。

### 2. 有序广播

<pre>
public abstract void sendOrderedBroadcast (Intent intent, String receiverPermission)

public abstract void sendOrderedBroadcast (Intent intent, 
					String receiverPermission, 
					BroadcastReceiver finalResultReceiver, 
					Handler scheduler, 
					int initialCode, 
					String initialData, 
					Bundle initialExtras)
</pre>

finalResultReceiver作为最末尾的receiver，可以得到前面一系列receiver的处理结果。通常应该提供自定义的receiver。<br/> 

scheduler一般设为null，说明使用context的主线程。<br/>

initialCode一般设为RESULT_OK，作为resultCode的初始值。<br/>

initialData一般设为null。作为resultData的初始值。<br/>

initialExtras一般设为null，作为resultExtras的初始值。<br/>


通过Context.sendOrderedBroadcast发送，receiver会按照android：priority所指定的优先级由大到小依次执行（android：priority范围为-1000~1000，数值越大优先级越高，默认优先级为0），由于是有序传播，可以实现如下效果：

- 终止传播：通过调用abortBroadcast，之后的receiver将接收不到该广播

- 向后继者传递数据：在onReceive中可以调用setResult、setResultCode、setResultData/setResultExtras设置信息，后继者可以通过getResultCode、getResultData、getResultExtra来获取信息。sendOrderedBroadcast中的参数initialCode、initialData、initialExtras提供了初始值。

**注意**：当静态注册与动态注册使用了相同的优先级（priority）时，动态注册的receiver处于更优先的位置。

## 安全方面的考虑

### 1. 隐患

- 其他APP可能会针对性的发出与当前APP中intent-filter相匹配的广播，导致当前APP不断受到广播并处理。

- 其他APP可能注册与当前APP一致的intent-filter用于接收广播，从而截获了广播的具体信息

### 2. 措施

- 如果receiver是用于同一APP内部的，则直接将其exported设为false

- 在发送广播时，使用含有permission的版本

- 在动态注册receiver时，使用含有permission的版本

- 在静态注册receiver时，在xml中增加permission字段

- 在发送广播时，可以指定receiver的包名。通过intent.setPackage(packageName)

### 3. 更便捷的方式——使用LocalBroadcastManager

- 获取单例
<pre>
static LocalBroadcastManager getInstance(Context context);
</pre>

- 注册
<pre>
void registerReceiver(BroadcastReceiver receiver, IntentFilter filter);
</pre>

- 注销
<pre>
void unregisterReceiver(BroadcastReceiver receiver);
</pre>

- 发送广播
<pre>
boolean	sendBroadcast(Intent intent);
</pre>

- 发送广播（同步发送，会阻塞直至所有相关receiver执行完毕onReceive并返回）
<pre>
void sendBroadcastSync(Intent intent);
</pre>

## 注册方式

### 1. 静态注册

通过xml文件的形式进行注册，此种方式注册的receiver会在APP运行期间一直存在，当APP被kill后就接收不到了。此种方式更为常用。

通过静态注册方式注册的receiver，其onReceive（Context context， Intent intent）中的context为ReceiverRestrictedContext（Context含有的bindService和registerReceiver函数被禁用）

**注意**：当BroadcastReceiver作为内部类被实现，同时又使用了静态注册的方式，那么该内部类必须声明为“**public static**”

**注意**：从4.0开始，需要至少启动一次APP后，静态注册的receiver才算注册完毕

### 2. 动态注册

通过代码的方式进行注册，BroadcastReceiver可实现为内部类或一般的外部类。

一般情况下，可以在onResume注册receiver，在onPause中注销receiver（少数情况下，也可以在onCreate中注册，在onDestory中注销）

通过动态注册方式注册的receiver，其onReceive（Context context， Intent intent）中的context为Activity的Context。

### 3. LocalBroadcastManager的动态注册

如果广播只是在APP内部进行收发，那么更高效和安全的方式是使用LocalBroadcastManager。这种方式只能够进行动态注册。

采用此种方式注册的receiver，其onReceive（Context context， Intent intent）中的context为Application的Context。

## 生命周期

### receiver的生命周期

receiver对象只有在onReceive的调用期间是有效的，是**“活跃”**的，一旦从onReceive中返回，那么系统将认为该对象已经结束，变为**“不活跃”**状态。

任何需要异步的操作都不应该放在onReceive中，因为当异步操作结束时，receiver可能已经处于“不活跃”状态，不能保证对象是否还存在。

**注意**：不能够在onReceive中显示对话框，可行的替代方案是使用NotificationManager相关功能。

**注意**：不能够在onReceive中绑定服务（bindService），可行的替代方案是通过startService发送命令。

### process的生命周期

正在执行receiver中onReceive的代码的process被认为是“前台进程”，它会一直保持运行（除非遇到非常极端的内存方面的压力才会被kill，这一般不会出现）

一旦从onReceive中返回，receiver变为“不活跃”状态，它所在的process的重要性将取决于运行在该process中的其他组件。如果这个process没有其他组件在运行（比如一个APP，用户近期没有与之交互过），那么系统将认为这是一个“空process”，会在合适的时机kill掉它。

产生的问题：如果在onReceive中产生一个thread，然后返回，整个进程、包括新产生的thread，都不认定为“不活跃”的，存在被kill的危机。解决的方式是与service相结合，在onReceive中启动一个Service，让Service去做具体的工作，由于Service的存在，当前的process的重要性取决于Service的状态，只要Service执行的工作未完成，它就一直是“活跃”的，就不会被kill。

## 其他注意事项

如果希望在receiver中的onReceive中开启一个Activity，则必须增加FLAG\_ACTIVITY\_NEW\_TASK标记。

onReceive必须在10秒钟内执行完毕，否则会产生ANR（Application Not Response）。如确实需要进行耗时的操作，可以通过启动一个Service的方式进行。

与广播相关的Intent的FLAG：

<pre>
FLAG_EXCLUDE_STOPPED_PACKAGES            （不再通知process被终止的receiver，默认行为）

FLAG_INCLUDE_STOPPED_PACKAGES            （仍然通知process被终止的receiver）
</pre>

从3.1开始，如果静态注册的APP退出后，不一定能够收到广播。<br/>
因为3.1开始系统增加了对APP是否处于运行状态的跟踪。在发送广播时，系统默认增加了FLAG\_EXCLUDE\_STOPPED\_PACKAGES的flag,导致即使是静态注册的receiver，当其所在process退出后，同样无法接收到广播。

对于自定义的广播，可以修改这种行为，使静态注册的receiver在process被结束后依然可以收到广播，方法就是在intent中将FLAG\_EXCLUDE\_STOPPED\_PACKAGES改写为FLAG\_INCLUDE\_STOPPED\_PACKAGES。

对于系统广播，则无能为力了。

## 常见系统广播

### ACTION\_TIME\_TICK

当前时间变化，每分钟广播一次。只能使用Context.registerReceiver()动态注册，静态注册无效。

值： "android.intent.action.TIME_TICK"


### ACTION\_TIME\_CHANGED

系统时间被设置。

值: "android.intent.action.TIME_SET"


### ACTION\_TIMEZONE\_CHANGED

时区被修改。带有extra：time-zone

值: "android.intent.action.TIMEZONE_CHANGED"


### ACTION\_BOOT\_COMPLETED

系统启动完成。可用进行一些初始化工作，比如安装alarm等。

权限：RECEIVE\_BOOT\_COMPLETED

值: "android.intent.action.BOOT_COMPLETED"


### ACTION\_PACKAGE\_ADDED

新的APP被安装。（新安装的APP不会受到此广播）

Data：新APP的包名

可能包含的Extras:

- EXTRA_UID 新APP的UID.
- EXTRA_REPLACING 是否是重装或升级。如果这个广播紧跟在ACTION_PACKAGE_REMOVED之后，并且作用的是同一个包，那么这个值为true

值: "android.intent.action.PACKAGE_ADDED"

### ACTION\_PACKAGE\_CHANGED

已安装的APP被改动，比如禁用或使能了某个组件。

Data： 包名

Extras：

- EXTRA_UID 包的UID
- EXTRA\_CHANGED\_COMPONENT\_NAME\_LIST 包含了被修改的组件的类名（或包名本身）
- EXTRA\_DONT\_KILL\_APP 布尔量，是否覆盖重启APP的默认action（待确认）

值: "android.intent.action.PACKAGE_CHANGED"

### ACTION\_PACKAGE\_REMOVED

已安装的APP被卸载。

Data：被卸载的APP的包名。

Extras：

- EXTRA_UID 被卸载APP的uid
- EXTRA_DATA_REMOVED 如果整个APP（包含代码和数据）被卸载，其值被设为true
- EXTRA_REPLACING 是否是重装或升级。如果这个广播后面紧跟着ACTION_PACKAGE_ADDED之后，并且作用的是同一个包，那么这个值为true

值: "android.intent.action.PACKAGE_REMOVED"


### ACTION\_PACKAGE\_RESTARTED

用户重启了这个APP，它的所有process被kill，所有与它相关的运行时状态被移除（包括process、alarm、notification等）。被重启的APP收不到这个广播。

Data：被重启APP的包名

Extras：

- EXTRA_UID 被重启APP的uid

值: "android.intent.action.PACKAGE_RESTARTED"


### ACTION\_PACKAGE\_DATA\_CLEARED

用户清空了APP的数据。这需要发生在ACTION\_PACKAGE\_RESTARTED之前。在擦除该APP所有持久化数据之后，本广播被发出。被清空的APP收不到此广播。

Data：被清空数据的APP的包名

Extras：

- EXTRA_UID APP的uid

值: "android.intent.action.PACKAGE\_DATA\_CLEARED"


### ACTION\_UID\_REMOVED

UID被从系统移除。UID被以EXTRA_UID为键存储在extras中。

值: "android.intent.action.UID_REMOVED"

### ACTION\_BATTERY\_CHANGED

粘性广播被声明为过时的，此处待验证。

### ACTION\_POWER\_CONNECTED

连接外部电源。

值: "android.intent.action.ACTION_POWER_CONNECTED"

### ACTION\_POWER\_DISCONNECTED

外部电源被移除。

值: "android.intent.action.ACTION_POWER_DISCONNECTED"
