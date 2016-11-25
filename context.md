#Context

***

## 分类

### Application的Context

Application对象以单例的形式出现，代表正在运行的APP，Application继承了ContextWrapper，所以可以认为它也是一个Context，此处将其称为“Application的Context”。

获取方式：

- 在Activity或Service中，可通过调用getApplication()函数获取。

- 可以通过context.getApplicationContext()获取。

### Activity、Service的Context

Activity和Service同样都继承于ContextWrapper，所以也可以认为它们是Context。

### BroadcastReceiver的Context

BroadcastReceiver本身不是Context，其内部也不含有Context，但在onReceive(Context context, Intent intent)中有context参数。这个context随着receiver的注册方式的不同而不同：

- 静态注册：context为ReceiverRestrictedContext，bindService和registerReceiver被禁用

- 动态注册：context为Activity的context

- LocalBroadcastManager的动态注册： context为Application的context

### ContentProvider的Context

ContentProvider本身不是Context，但可以通过getContext()获取一个context对象（该对象代表的是当前provider运行的context）。具体来讲：

- 如果provider和调用者在同一个process中，context就是Application的context

- 如果provider和调用者分属不同的进程，getContext将创建一个新的context代表此provider所运行的包。

getContext()必须在onCreate调用之后才可用，在构造器中调用将返回null。

## 能力

<table border="true">
<tr>
<th></th>
<th>Application</th>
<th>Activity</th>
<th>Service</th>
<th>Content Provider</th>
<th>Broadcast Receiver(静态)</th>
</tr>

<tr>
<th>显示对话框</th>
<td>No</td>
<td>Yes</td>
<td>No</td>
<td>No</td>
<td>No</td>
</tr>

<tr>
<th>启动Activity</th>
<td>No<sup>1</sup></td>
<td>Yes</td>
<td>No<sup>1</sup></td>
<td>No<sup>1</sup></td>
<td>No<sup>1</sup></td>
</tr>

<tr>
<th>填充布局</th>
<td>No<sup>2</sup></td>
<td>Yes</td>
<td>No<sup>2</sup></td>
<td>No<sup>2</sup></td>
<td>No<sup>2</sup></td>
</tr>

<tr>
<th>启动Service</th>
<td>Yes</td>
<td>Yes</td>
<td>Yes</td>
<td>Yes</td>
<td>Yes</td>
</tr>

<tr>
<th>绑定Service</th>
<td>Yes</td>
<td>Yes</td>
<td>Yes</td>
<td>Yes</td>
<td>No</td>
</tr>

<tr>
<th>发送Broadcast</th>
<td>Yes</td>
<td>Yes</td>
<td>Yes</td>
<td>Yes</td>
<td>Yes</td>
</tr>

<tr>
<th>注册Broadcast-Receiver</th>
<td>Yes</td>
<td>Yes</td>
<td>Yes</td>
<td>Yes</td>
<td>No<sup>3</sup></td>
</tr>

<tr>
<th>加载资源值</th>
<td>Yes</td>
<td>Yes</td>
<td>Yes</td>
<td>Yes</td>
<td>Yes</td>
</tr>

</table>


**注意**：

- No<sup>1</sup>表示Application的Context确实可以启动一个Activity，但是它需要创建一个新的task，会造成APP中存在不标准的回退栈，不推荐。
- No<sup>2</sup>表示这是非法的，填充虽然可以完成，但使用的系统默认的theme，而非APP的theme。
- No<sup>3</sup>在4.2以上，如果receiver是null（用于粘性广播，已被标注为过时），是允许的。

综上：与UI相关的功能只能由Activity的Context去处理。


## 注意事项

当需要保存一个context的引用时，如果它超过了你的Activity或Service的生命周期（即便只是暂时的），需要保存Application的context。

比较典型的：单例。


错误的例子：
<pre><code>
public class CustomManager{
	private static CustomManager sInstance;

	private Context mContext;
	private CustomManager(Context context){
		mContext = context;
	}

	public static CustomManager getInstance(Context context){
		if( sInstance == null ){
			sInstance = new CustomManager(context);
		}
		return sInstance;
	}
}
</code></pre>

原因：如果传入的context是一个Activity，则由于它在单例的实现中被引用，导致该Activity对象及其所引用的对象永远不能被垃圾回收，有内存泄漏的风险。


更好的实现：
<pre><code>
public class CustomManager{
	private static CustomManager sInstance;

	private Context mContext;
	private CustomManager(Context context){
		mContext = context;
	}

	public static CustomManager getInstance(Context context){
		if( sInstance == null ){
			sInstance = new CustomManager(context.getApplicationContext());
		}
		return sInstance;
	}
}
</code></pre>

另一个比较典型的例子：在后台线程或一个等待的Handler中如果需要保存Context的引用，也请使用Application的context。