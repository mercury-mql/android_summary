# 窗口 Activity
***
## 基本应用

### 1. 生命周期

![](images/activity_lifecycle.png)

正常情况下的生命周期：

可见性： onStop ——> onRestart ——> onStart ——> onResume ——> onPause ——> onStop

焦点（交互性）： onPause ——> onResume ——> onPause

注意：onRestoreInstanceState在onStart之后，onPostCreate之前被调用

当系统内存吃紧时，系统会销毁长期被停止（stop）的窗口对象，这些对象往往存在于回退栈的底层，系统销毁时会销毁窗口对象本身（onDestroy）和该窗口对象的状态。当此窗口再次回到栈顶时，系统不仅需要重新构建窗口对象（通过onCreate），同时也要恢复窗口的状态（这就需要在窗口被kill前将系统状态保存下来），onSaveInstanceState就提供了一个保存窗口状态的时机。





                                                          
