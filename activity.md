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

### android:name
唯一必须设置的属性，标识了Activity的类，其值可以为：
- 相对类名（相对于<manifest/>中的package的属性值），指定相对类名时，以“.”开头，类由package+android：name来确定

- 绝对类名（完整的类名）

### android：label
标题文本

### android：icon
标题图像

**如果未设置Activity的android：label或android：icon，会使用Application的同名属性；如Application也未设置，会使用系统的默认值**

### android：screenOrientation
Activity显示的方向。在Java代码中，可使用
<pre><code>
void setRequestedOrientation(int requestedOrientation)
</code></pre>

### android:configChanges
默认情况下，配置变化后，Activity会自己通过销毁和重建的方式来出来这些变化。但可以通过android：configChanges属性来改变这种情况。

可以通过android：configChanges来指定一项或多项配置，那么当这些配置发生变化时，系统会回调<pre><code>
public void onConfigurationChanged (Configuration newConfig)
</code></pre>

### android:enabled
是否允许该Activity被实例化，默认值为true。
如设为false，则该Activity不能被实例化。

### android：excludeFromRecents
是否将该Activity排除在最近应用之外。默认值为false。
如设为true，则该Activity不会显示在最近应用的列表中。

### android：exported
是否允许其他APP访问该Activity，默认为true。其值在对于APP内部的访问无效果。
**即使exported被设为true，也必须指定一个Action才能实现“隐式访问”的效果**

### android：hardwareAccelerated
硬件加速。默认为false。

### android：multiprocess
该Activity的实例能否运行在多个process中。默认值是false。

默认情况下，Activity实例运行在定义它的APP的进程中，因此所有的实例运行在同一个process中。然而，当android：multiprocess被设为true时，Activity的实例运行在启动它的组件的process中（也就是说同一Activity的不同实例运行在不同的process中）

### android：noHistory
该Activity是否不进入回退栈，默认为false。
如设为true，那么该Activity不会进入回退栈，当用户离开它时（比如通过startActivity启动一个其他的Activity）时，它的finish()被调用，用户无法返回到该Activity，它的onActivityResult永远不会得到调用。


### android：parentActivityName
该Activity的逻辑parent，即当用户点击Action Bar中的UP按钮时，系统会根据这个属性的值来决定哪一个Activity将会被启用。

### android：permission
该Activity的权限。如果一个组件通过startActivity或startActivityForResult试图启动这个Activity时，该组件需要在它的manifest中使用<uses-permission\>声明这一权限，否则此Activity不会受到intent。
如果android：permission未被设置，它会使用<application\>的同名属性，如果<application\>也未设置该属性，那么该Activity没有被权限保护。


### android：process
指明此Activity应该运行在的process。
通常情况下，一个APP的所有组件都运行在同一个process中，这个process的名称就是APP的名称。

android：process这个属性就允许你修改默认的process名称，从而使APP的各个组件能够运行在不同的process中。android：process的值有两种情况：

- 以“：”开头，该process是该APP私有的，它在合适的时间创建，被声明的Activity就运行在这个process中。

- 不以“：”开头，该process是全局的，这意味着不同APP中的组件可以共享这一process。


### android：stateNotNeeded

Whether or not the activity can be killed and successfully restarted without having saved its state — "true" if it can be restarted without reference to its previous state, and "false" if its previous state is required. The default value is "false".
Normally, before an activity is temporarily shut down to save resources, its onSaveInstanceState() method is called. This method stores the current state of the activity in a Bundle object, which is then passed to onCreate() when the activity is restarted. If this attribute is set to "true", onSaveInstanceState() may not be called and onCreate() will be passed null instead of the Bundle — just as it was when the activity started for the first time.

A "true" setting ensures that the activity can be restarted in the absence of retained state. For example, the activity that displays the home screen uses this setting to make sure that it does not get removed if it crashes for some reason.

### android：theme

A reference to a style resource defining an overall theme for the activity. This automatically sets the activity's context to use this theme (see setTheme(), and may also cause "starting" animations prior to the activity being launched (to better match what the activity actually looks like).
If this attribute is not set, the activity inherits the theme set for the application as a whole — from the <application> element's theme attribute. If that attribute is also not set, the default system theme is used. For more information, see the Styles and Themes developer guide.

### android：uiOptions



### android：alwaysRetainTaskState

Whether or not the state of the task that the activity is in will always be maintained by the system — "true" if it will be, and "false" if the system is allowed to reset the task to its initial state in certain situations. The default value is "false". This attribute is meaningful only for the root activity of a task; it's ignored for all other activities.
Normally, the system clears a task (removes all activities from the stack above the root activity) in certain situations when the user re-selects that task from the home screen. Typically, this is done if the user hasn't visited the task for a certain amount of time, such as 30 minutes.

However, when this attribute is "true", users will always return to the task in its last state, regardless of how they get there. This is useful, for example, in an application like the web browser where there is a lot of state (such as multiple open tabs) that users would not like to lose.
### android：clearTaskOnLaunch

Whether or not all activities will be removed from the task, except for the root activity, whenever it is re-launched from the home screen — "true" if the task is always stripped down to its root activity, and "false" if not. The default value is "false". This attribute is meaningful only for activities that start a new task (the root activity); it's ignored for all other activities in the task.
When the value is "true", every time users start the task again, they are brought to its root activity regardless of what they were last doing in the task and regardless of whether they used the Back or Home button to leave it. When the value is "false", the task may be cleared of activities in some situations (see the alwaysRetainTaskState attribute), but not always.

Suppose, for example, that someone launches activity P from the home screen, and from there goes to activity Q. The user next presses Home, and then returns to activity P. Normally, the user would see activity Q, since that is what they were last doing in P's task. However, if P set this flag to "true", all of the activities on top of it (Q in this case) were removed when the user pressed Home and the task went to the background. So the user sees only P when returning to the task.

If this attribute and allowTaskReparenting are both "true", any activities that can be re-parented are moved to the task they share an affinity with; the remaining activities are then dropped, as described above.

### android：finishOnTaskLaunch
Whether or not an existing instance of the activity should be shut down (finished) whenever the user again launches its task (chooses the task on the home screen) — "true" if it should be shut down, and "false" if not. The default value is "false".
If this attribute and allowTaskReparenting are both "true", this attribute trumps the other. The affinity of the activity is ignored. The activity is not re-parented, but destroyed.

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










                                                          
