# 窗口 Activity

***

## 1. 生命周期

![](images/activity_lifecycle.png)

正常情况下的生命周期：

可见性： onStop ——> onRestart ——> onStart ——> onResume ——> onPause ——> onStop

焦点（交互性）： onPause ——> onResume ——> onPause


### onCreate

完成绝大多数初始化工作。

如果直接在这里调用finish(),将跳过其他生命周期方法（onStart、onResume、onPause、onStop之类），直接执行onDestroy方法。

### onResume

可用于：

- 开启动画
- 获取独占型的设备（如Camera）
- 注册BroadcastReceiver


### onPause

当Activity A处于栈顶位置，此时新建Activity B（B将成为栈顶，压在A上方），则在A的onPause执行完毕并返回后，B的创建工作才开始，所以不要在onPause中做耗时的工作。

可用于：

- 关闭动画
- 结束其他消耗CPU的动作
- 释放独占型的设备（如Camera）
- 注销BroadcastReceiver
- 保存持久化信息
- 调用isFinishing()来判断当前Activity是否要被销毁（由Back按键或调用finish导致）

这里要额外提一下，虽然可以在onSaveInstanceState中保存窗口状态，但onSaveInstanceState只在系统因内存紧张等资源的原因而不得已kill当前窗口对象的时候才会调用，而由用户主动按Back间退出程序或程序代码调用finish的情况下，onSaveInstanceState不会得到调用，故存在丢失信息的风险。

当给用户呈现出“原地编辑（edit in place）”的模式时，onPause需要保存当前Acitivity正在编辑的持久化数据，从而保证如果不满足onSaveInstanceState调用条件的情况下，此类数据的更改不会丢失。

常见的做法是：onSaveInstanceState保存按实例（per-instance）的信息，而onPause用于保存全局性的持久化数据（比如在ContentProvider中的，file中的）


### onDestroy

做最后资源清理的地方。有两种调用场景：

- 系统资源不足，需要临时性的kill当前Activity
- 用户通过Back键退出本Activity，或代码中主动调用finish

可以在onPause中调用isFinishing来判断究竟是哪一种场景

注意：此函数不应用来保存数据。


***

## 2. 几个重要的函数

### onRestoreInstanceState
onRestoreInstanceState在onStart之后，onPostCreate之前被调用。它的默认实现会恢复在onSaveInstanceState默认实现中保存的UI控件的状态，所以如果要覆盖该方法，一定要调用super.onRestoreInstanceState。但一般情况下，不会覆盖该方法。

### onSaveInstanceState

当系统内存吃紧时，系统会销毁长期被停止（stop）的窗口对象，这些对象往往位于回退栈的底层，系统销毁这些窗口对象时，不仅会销毁窗口对象本身（onDestroy），同时也销毁该窗口对象的状态。当此窗口再次回到栈顶时，系统不仅需要重新构建窗口对象（通过onCreate），同时也要恢复窗口的状态（这就需要在窗口被kill前将系统状态保存下来），onSaveInstanceState就提供了一个保存窗口状态的时机。

onSaveInstanceState并不属于生命周期的方法，这也就是说当onPause、onStop等方法被调用时不一定伴随着onSaveInstanceState的调用。比如当一个新的Activity被生成，并被放入回退栈栈顶时，原来处于栈顶位置的Activity一定会调用onPause和onStop，但是不会调用onSaveInstanceState，因为此窗口没有被销毁。此外当用户通过BACK按键退出Activity或程序调用finish时，Activity会经历onPause ——> onStop ——> onDestory的过程，也不会调用onSaveInstanceState，这是应为Activity是被主动销毁的，不是系统因内存紧张而kill掉的，不满足onSaveInstance调用的条件。

onSaveInstanceState的默认实现会将Activity中有android：id的UI控件的状态保存下来，所以如果要覆盖onSaveInstanceState时，需要调用super.onSaveInstanceState。

如果被onSaveInstanceState会在onStop之前被调用，但与onPause之间谁先谁后并没有固定的顺序。

被onSaveInstanceState保存下来的状态（Bundle对象）可在onCreate或onRestoreInstanceState中恢复，更常见的是在onCreate中。


### finish

结束当前Activity。用户按Back键时会调用onBackPressed函数，该函数的默认实现也是调用了finish。

finish调用发生在onPause之前，所以可以在onPause中通过isFinishing进行判断。

finish会把ActivityResult(默认为RESULT_CANCELED)通过onActivityResult回传给任何启动它的Activity。

### setResult

<pre>
public final void setResult (int resultCode)
public final void setResult (int resultCode, Intent data)
</pre>

调用此函数可以为当前Activity设置result，result会被回传给当前Activity的调用者。

intent中可以设置<br/>
**Intent.FLAG\_GRANT\_READ\_URI\_PERMISSION** <br/>
和<br/> 
**Intent.FLAG\_GRANT\_WRITE\_URI\_PERMISSION**. <br/>
如果设置了这样的标志，收到这个result的Actvity（简称为接收Activity）将会被赋予访问（读写）Intent中所包含的URI的权限。这种访问权限将会一直保持到“接受Activity”结束（如果接收Activity被kill或其他临时性的释放，此种访问权限依然能够保持）。通过这种方式赋予的权限会添加到“接收Activity”目前持有的权限集中。

### onActivityResult

<pre>protected void onActivityResult (int requestCode, int resultCode, Intent data)</pre>

当此Activity启动的子Activity结束时，被回调。从而获取到启动时的请求码（requestCode），子Activity返回的结果码（resultCode，由setResult设置），以及其他额外的数据(intent,由setResult设置）。

此调用发生在onResume之前。

如果设置android：noHistory为true，那么此方法永远不会被调用。


### startActivityForResult

<pre>
public void startActivityForResult (Intent intent, int requestCode)
public void startActivityForResult (Intent intent, int requestCode, Bundle options)
</pre>

传入的请求码（requestCode）颇有讲究：

- requestCode>=0，则会导致后来的onActivityResult被调用
- requestCode<0，相当于调用了startActivity（intent），将要启动的Activity将不会作为当前Activity的子Activity。


如果要启动的Activity使用了singleTask的启动模式（或设置FLAG\_ACTIVITY\_NEW\_TASK标志等任何能使要启动的Activity运行在另一个任务中的手段），只要它运行在另一个任务中,onActivityResult将立即返回，resultCode为RESULT_CANCELLED。

有一种特殊情况，当在onCreate或onResume中以requestCode>=0的形式调用startActivityForResult时，则当前窗口不会显示，而直接显示子窗口，从而避免了闪烁。

***

## 3. Intent

### Intent中包含的信息

#### 3.1 Component name

可以直接在Intent中指定要访问的对象，这可用于显式访问。有这么几种方式：

##### 方式一：直接指定class

<pre><code>
Intent intent = new Intent(MainActivity.this, TargetActivity.class)
</code></pre>

##### 方式二：使用setClass

函数原型：
<pre><code>
Intent setClass(Context packageContext, Class<?> cls)
</code></pre>
等价于Intent的另一个构造函数：
<pre><code>
public Intent(Context packageContext, Class<?> cls)
</code></pre>

用法：
<pre><code>
Intent intent = new Intent();
intent.setClass(MainActivity.this, TargetActivity.class);
</code></pre>

##### 方式三：使用setClassName

函数原型：
<pre><code>
Intent setClassName(Context packageContext, String className)

Intent	setClassName(String packageName, String className)
</code></pre>

其中，className指的是“类全名”（即包名+类名）

##### 方式四：使用setComponent

函数原型：
<pre><code>
Intent setComponent(ComponentName component)
</code></pre>

需要首先创建ComponentName对象，使用：
<pre><code>
ComponentName(String pkg, String cls)

ComponentName(Context pkg, String cls)

ComponentName(Context pkg, Class<?> cls)

ComponentName(Parcel in)
</code></pre>

#### 3.2 Action
动作。本质是一个字符串。
#### 3.3 Category
#### 3.4 Data
#### 3.5 Extra
#### 3.6 Flags

### 访问窗口

### 过滤机制

***

## 4. Activity间传递数据的其他方式（除Intent之外）

### 静态变量

### 剪贴板

### 全局对象

***

## 5. Activity的常见属性

***

## 6. Activity的启动模式与Flags

***

## 7. Activity常用事件

***

## 8. 几个常用效果

### 全屏显示

### 标题栏定制

### 点击两次关闭窗口

### Splash

### 关闭所有窗口

### 窗口截屏

***

## 9. 显示系统窗口










                                                          
