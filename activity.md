# 窗口 Activity
***
## 基本知识点

### 1. 生命周期

![](images/activity_lifecycle.png)

正常情况下的生命周期：

可见性： onStop ——> onRestart ——> onStart ——> onResume ——> onPause ——> onStop

焦点（交互性）： onPause ——> onResume ——> onPause


#### onCreate

完成绝大多数初始化工作。

如果直接在这里调用finish(),将跳过其他生命周期方法（onStart、onResume、onPause、onStop之类），直接执行onDestroy方法。

#### onResume

可用于：

- 开启动画
- 获取独占型的设备（如Camera）
- 注册BroadcastReceiver


#### onPause

当Activity A处于栈顶位置，此时新建Activity B（B将成为栈顶，压在A上方），则在A的onPause执行完毕并返回后，B的创建工作才开始，所以不要在onPause中做耗时的工作。

可用于：

- 关闭动画
- 结束其他消耗CPU的动作
- 释放独占型的设备（如Camera）
- 注销BroadcastReceiver
- 保存窗口状态

这里要额外提一下，虽然可以在onSaveInstanceState中保存窗口状态




### 2. 几个重要的函数

#### onRestoreInstanceState相关
onRestoreInstanceState在onStart之后，onPostCreate之前被调用。它的默认实现会恢复在onSaveInstanceState默认实现中保存的UI控件的状态，所以如果要覆盖该方法，一定要调用super.onRestoreInstanceState。但一般情况下，不会覆盖该方法。

#### onSaveInstance相关

当系统内存吃紧时，系统会销毁长期被停止（stop）的窗口对象，这些对象往往位于回退栈的底层，系统销毁这些窗口对象时，不仅会销毁窗口对象本身（onDestroy），同时也销毁该窗口对象的状态。当此窗口再次回到栈顶时，系统不仅需要重新构建窗口对象（通过onCreate），同时也要恢复窗口的状态（这就需要在窗口被kill前将系统状态保存下来），onSaveInstanceState就提供了一个保存窗口状态的时机。

onSaveInstanceState并不属于生命周期的方法，这也就是说当onPause、onStop等方法被调用时不一定伴随着onSaveInstanceState的调用。比如当一个新的Activity被生成，并被放入回退栈栈顶时，原来处于栈顶位置的Activity一定会调用onPause和onStop，但是不会调用onSaveInstanceState，因为此窗口没有被销毁。此外当用户通过BACK按键退出Activity或程序调用finish时，Activity会经历onPause ——> onStop ——> onDestory的过程，也不会调用onSaveInstanceState，这是应为Activity是被主动销毁的，不是系统因内存紧张而kill掉的，不满足onSaveInstance调用的条件。

onSaveInstanceState的默认实现会将Activity中有android：id的UI控件的状态保存下来，所以如果要覆盖onSaveInstanceState时，需要调用super.onSaveInstanceState。

如果被onSaveInstanceState会在onStop之前被调用，但与onPause之间谁先谁后并没有固定的顺序。

被onSaveInstanceState保存下来的状态（Bundle对象）可在onCreate或onRestoreInstanceState中恢复，更常见的是在onCreate中。







                                                          
