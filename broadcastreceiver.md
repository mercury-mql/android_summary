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

## 注册方式

### 1. 静态注册

#### 生命周期

#### Context

### 2. 动态注册

#### 生命周期

#### Context

### 3. LocalBroadcastManager的动态注册

#### 生命周期

#### Context


## 其他注意事项

## 常见系统广播

