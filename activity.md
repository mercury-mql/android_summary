# 窗口 Activity

***

写在前面：每一个Android APP在运行时都会创建一个Task，每个Task都包含一个栈结构（被称为“回退栈”）。

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

### （1）android:name
唯一必须设置的属性，标识了Activity的类，其值可以为：
- 相对类名（相对于<manifest/>中的package的属性值），指定相对类名时，以“.”开头，类由package+android：name来确定

- 绝对类名（完整的类名）

### （2）android：label
标题文本

### （3）android：icon
标题图像

**如果未设置Activity的android：label或android：icon，会使用Application的同名属性；如Application也未设置，会使用系统的默认值**

### （4）android：screenOrientation
Activity显示的方向。在Java代码中，可使用
<pre><code>
void setRequestedOrientation(int requestedOrientation)
</code></pre>

### （5）android:configChanges
默认情况下，配置变化后，Activity会自己通过销毁和重建的方式来出来这些变化。但可以通过android：configChanges属性来改变这种情况。

可以通过android：configChanges来指定一项或多项配置，那么当这些配置发生变化时，系统会回调<pre><code>
public void onConfigurationChanged (Configuration newConfig)
</code></pre>

### （6）android:enabled
是否允许该Activity被实例化，默认值为true。
如设为false，则该Activity不能被实例化。

### （7）android：excludeFromRecents
是否将该Activity排除在最近应用之外。默认值为false。
如设为true，则该Activity不会显示在最近应用的列表中。

### （8）android：exported
是否允许其他APP访问该Activity，默认为true。其值在对于APP内部的访问无效果。
**即使exported被设为true，也必须指定一个Action才能实现“隐式访问”的效果**

### （9）android：hardwareAccelerated
硬件加速。默认为false。

### （11）android：multiprocess
该Activity的实例能否运行在多个process中。默认值是false。

默认情况下，Activity实例运行在定义它的APP的进程中，因此所有的实例运行在同一个process中。然而，当android：multiprocess被设为true时，Activity的实例运行在启动它的组件的process中（也就是说同一Activity的不同实例运行在不同的process中）

### （12）android：noHistory
该Activity是否不进入回退栈，默认为false。
如设为true，那么该Activity不会进入回退栈，当用户离开它时（比如通过startActivity启动一个其他的Activity）时，它的finish()被调用，用户无法返回到该Activity，它的onActivityResult永远不会得到调用。


### （13）android：parentActivityName
该Activity的逻辑parent，即当用户点击Action Bar中的UP按钮时，系统会根据这个属性的值来决定哪一个Activity将会被启用。

### （14）android：permission
该Activity的权限。如果一个组件通过startActivity或startActivityForResult试图启动这个Activity时，该组件需要在它的manifest中使用<uses-permission\>声明这一权限，否则此Activity不会受到intent。
如果android：permission未被设置，它会使用<application\>的同名属性，如果<application\>也未设置该属性，那么该Activity没有被权限保护。


### （15）android：process
指明此Activity应该运行在的process。
通常情况下，一个APP的所有组件都运行在同一个process中，这个process的名称就是APP的名称。

android：process这个属性就允许你修改默认的process名称，从而使APP的各个组件能够运行在不同的process中。android：process的值有两种情况：

- 以“：”开头，该process是该APP私有的，它在合适的时间创建，被声明的Activity就运行在这个process中。

- 不以“：”开头，该process是全局的，这意味着不同APP中的组件可以共享这一process。


### （16）android：stateNotNeeded
是否不需要保存状态。默认为false。

如设为true，该Activity的onSaveInstanceState永远不会被调用，onCreate中的Bundle永远为null。


### （17）android：theme
主题。

在代码中可使用setTheme。

如果该属性未设置，将会使用<application\>中的同名属性，如果<application\>也没有设置，将使用系统默认属性。



### （18）android：uiOptions

暂缺

### （19）android：alwaysRetainTaskState
该Activity所在的task的状态是否需要系统一直保持。默认为false。

当设为true时，无论用户离开该Activity所在的Task多长时间，系统都不会销毁此Task回退栈中的任何窗口。

**注意：这个属性只对Task中的根Activity有效，其他Activity即使设置了这个属性也会被忽略掉**

通常，系统会在某些情况下清理一个Task（也就是说移除其根Activity之上的所有窗口），这通常发生在用户一段时间内没有访问这个Task（比如30分钟）同时由从HOME screen中选择了这个APP的时候。

但是，当这个属性被设为true时，用户返回到这个task时，永远看到的是它最后的状态。

### （20）android：clearTaskOnLaunch

与android:alwaysRetainTaskState作用相反。决定当该APP从HOME screen被重新启动时，是否要将除根Activity之外的其他所有窗口出栈。默认为false。

当设为true时，无论何时，该task一旦被切入后台（哪怕是很短的时间），task的回退栈中除根Activity之外的所有Activity都被移除。那么，当用户从HOME screen中再次选择这个APP时，看到的永远是根Activity。

当设为false时，系统会在某些情况下清理一个Task（也就是说移除其根Activity之上的所有窗口），这通常发生在用户一段时间内没有访问这个Task（比如30分钟）同时由从HOME screen中选择了这个APP的时候。

**注意：这个属性只对根Activity有效，其他Activity即使设置了也会被忽略**

假如，如果从HOME screen启动一个Activity P（P为根Activity），然后转至Activity Q，然后用户按HOME键将此Task切入后台，当用户再次点击APP图标试图返回该Task时：

- 如果未设置android：clearTaskOnLaunch，用户将可能看到Q（除非用户过了很长时间才返回该Task，比如30分钟）

- 如果将android：clearTaskOnLaunch设为true，无论何时返回，用户都将看到P。

**注意：如果这个属性和android：allowTaskReparenting都设为true时，那么所有能够re-parented的Activity都会被转移到他们taskAffinity指定的task，剩下的将会被出栈。**


### （21）android：finishOnTaskLaunch
当用户再次启动task时，此Activity的实例是否被销毁。默认为false。

此属性与android：clearTaskOnLaunch类似，不过android：finishOnTaskLaunch针对的是一个Activity，而android：clearTaskOnLaunch针对的是整个task。

例如：目前回退栈中含有三个Activity，A-B-C，其中C为栈顶。如果C的android：finishOnTaskLaunch被设为true，那么当这个task被切入后台后，C将被销毁，栈中将剩下A-B，当用户再次返回到这个task时，将见到B。

**注意：此属性对根Activity无效。**
**注意：如果这个属性和allowTaskReparenting都被设为true，此属性优先，此Activity会被销毁而不被re-parent。**

### （22）android：taskAffinity

应与FLAG\_ACTIVITY\_NEW\_TASK配合使用。

每个Activity都有taskAffinity属性，指明了它希望进入的Task（即它希望使用的回退栈）。

如果没有指定，将使用<application\>的同名属性，如果<application\>也没有指定该属性，将使用Activity所在的**包名**作为默认值。

The task that the activity has an affinity for. Activities with the same affinity conceptually belong to the same task (to the same "application" from the user's perspective). The affinity of a task is determined by the affinity of its root activity.
The affinity determines two things — the task that the activity is re-parented to (see the allowTaskReparenting attribute) and the task that will house the activity when it is launched with the FLAG_ACTIVITY_NEW_TASK flag.

By default, all activities in an application have the same affinity. You can set this attribute to group them differently, and even place activities defined in different applications within the same task. To specify that the activity does not have an affinity for any task, set it to an empty string.

If this attribute is not set, the activity inherits the affinity set for the application (see the <application> element's taskAffinity attribute). The name of the default affinity for an application is the package name set by the <manifest> element.

### （23）android：allowTaskReparenting

Whether or not the activity can move from the task that started it to the task it has an affinity for when that task is next brought to the front — "true" if it can move, and "false" if it must remain with the task where it started.
If this attribute is not set, the value set by the corresponding allowTaskReparenting attribute of the <application> element applies to the activity. The default value is "false".

Normally when an activity is started, it's associated with the task of the activity that started it and it stays there for its entire lifetime. You can use this attribute to force it to be re-parented to the task it has an affinity for when its current task is no longer displayed. Typically, it's used to cause the activities of an application to move to the main task associated with that application.

For example, if an e-mail message contains a link to a web page, clicking the link brings up an activity that can display the page. That activity is defined by the browser application, but is launched as part of the e-mail task. If it's reparented to the browser task, it will be shown when the browser next comes to the front, and will be absent when the e-mail task again comes forward.

The affinity of an activity is defined by the taskAffinity attribute. The affinity of a task is determined by reading the affinity of its root activity. Therefore, by definition, a root activity is always in a task with the same affinity. Since activities with "singleTask" or "singleInstance" launch modes can only be at the root of a task, re-parenting is limited to the "standard" and "singleTop" modes. (See also the launchMode attribute.)

***

## 6. Activity的启动模式与Flags

### 6.1 四种创建模式（Task的模式）

通过android：launchMode指定

#### （1）standard

默认的启动模式。一个Activity可以有多个实例，每一个实例可以属于不同的task，一个task可以有同一个Activity的多个实例。

#### （2）singleTop

两类情况：

- 情况一：该Activity的实例刚好处于当前Task的回退栈的栈顶位置，直接使用该实例，同时调用其onNewIntent方法。

- 情况二：该Activity的实例不在当前Task的回退栈中，或者，虽在回退栈中但不处于栈顶位置，创建该Activity的实例，并将其压入回退栈


#### （3）singleTask

五类情况：

- 情况一：调用同一APP中的Activity，在当前Task的回退栈中不存在该Activity的实例，创建一个实例，并压入回退栈中

- 情况二：调用同一APP中的Activity，在当前Task的回退栈中存在该Activity的实例，将该实例之上的所有对象出栈（onDestory），并调用该实例的onNewIntent

- 情况三：调用不同APP中的Activity，如果该Activity要求的Task不存在，创建该Task，并且创建该Activity的实例，将其压入新创建的Task的栈顶

- 情况四：调用不同APP中的Activity，如果该Activity要求的Task已存在，同时在该Task的回退栈中没有该Activity的实例，那么首先切换到该Task，并创建该Activity的实例，将其压入此Task的回退栈中

- 情况五：调用不同APP中的Activity，如果该Activity要求的Task已经存在，同时在该Task的回退栈中含有该Activity的实例，那么首先切换到该Task，同时将该Activity实例之上的所有对象出栈（onDestory），接着调用该Activity实例的onNewIntent

#### （4）singleInstance

被声明为singleInstance的Activity，要求整个Task中只有一个该Activity的实例，不允许存在其他对象。

被声明为singleInstance的Activity，具有全局唯一性，整个系统中只有一个实例。

两种情况：

- 情况一：调用被声明为singleInstance的Activity（无论该Activity是否与调用者在同一个APP中），该Activity的实例在系统中不存在，创建一个新Task，并创建一个Activity实例，将该实例压入新建Task的回退栈。

- 情况二：调用被声明为singleInstance的Activity（无论该Activity是否与调用者在同一个APP中），如果该Activity在系统中已经存在，那么切换到其所在的Task（注意没有调用onNewIntent）

**注意：无论调用者和被声明为singleInstance的Activity是否在一个Task中，只要系统中没有该Activity的实例，一定会创建一个新任务。**

### 6.2 影响创建模式的Flags

#### （1）FLAG\_ACTIVITY\_SINGLE\_TOP

相当于singleTop的launchMode

#### （2）FLAG\_ACTIVITY\_CLEAR\_TOP

三种情况：

- 情况一：该Activity不在回退栈中，创建一个实例，压入回退栈

- 情况二：该Activity已经在回退栈中存在，它的launchMode为“standard”，同时Intent中没有指定FLAG\_ACTIVITY\_SINGLE\_TOP，那么将会把此Activity和在它之上的所有Activity对象全部出栈（onDestroy）。然后新建一个该Activity的实例，并压入回退栈。

- 情况三：该Activity已经在回退栈中存在，同时它不满足情况二中的描述，那么在它之上的所有Activity对象被出栈（onDestroy），而它本身被复用，它的onNewIntent被调用。

#### （3）FLAG\_ACTIVITY\_NEW\_TASK

一般与android：taskAffinity配合使用。


当使用FLAG\_ACTIVITY\_NEW\_TASK启动某个Activity时，系统会寻找或创建一个Task来放置此Activity，具体来讲就是根据其android：taskAffinity的值来匹配，有两种情况：

- 情况一： 如果能够找到一个Task与该Activity中android：taskAffinity的设定值匹配，那么会切换到该Task，并创建一个Activity的实例，压入其回退栈。

- 情况二：如果在系统中找不到能够与android：taskAffinity的设定值相匹配的Task，那么首先创建一个Task，将此Task的affinity设置为该Activity的android：taskAffinity属性要求的值，同时创建一个Activity的实例，压入到此Task的回退栈中。

**注意：在同一个APP中，当所有的Activity都使用默认值，或者android：taskAffinity被指定了相同的值时，此标志不起作用。**

**注意：如果在BroadcastReceiver的onReceive中需要启动一个Activity时，务必在Intent中加入FLAG\_ACTIVITY\_NEW\_TASK标志。**

#### （4）FLAG\_ACTIVITY\_CLEAR\_TASK

必须配合FLAG\_ACTIVITY\_NEW\_TASK标志使用。

当用含有这个标志的Intent启动某个Activity时，会将当前Task的回退栈中的所有Activity对象销毁（onDestory），然后重新创建此Activity，将其作为新的Task的根Activity。

#### （5）FLAG\_ACTIVITY\_REORDER\_TO\_FRONT

调整Activity在回退栈中的顺序。

当使用此标志启动某一Activity时，如果在回退栈中有它的实例，那么会把它调至栈顶，同时调用它的onNewIntent方法。

例：
回退栈中有四个Activity对象，A-B-C-D，其中D为栈顶。此时用FLAG\_ACTIVITY\_REORDER\_TO\_FRONT启动B，那么回退栈变为A-C-D-B，同时B的onNewIntent方法被调用。

**注意：如果与FLAG\_ACTIVITY\_CLEAR\_TOP同时被设置，那么FLAG\_ACTIVITY\_REORDER\_TO\_FRONT将不起作用。**

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



## 10. 疑问
1、task与process的关系？
2、FLAG\_ACTIVITY\_CLEAR\_TASK是否与原来是同一个task？










                                                          
