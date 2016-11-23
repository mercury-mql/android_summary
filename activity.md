# 窗口 Activity
***
## 基本应用

### 1. 生命周期

![](images/activity_lifecycle.png)

正常情况下的生命周期：

可见性： onStop ——> onRestart ——> onStart ——> onResume ——> onPause ——> onStop

焦点（交互性）： onPause ——> onResume ——> onPause

注意：onRestoreInstanceState在onStart之后，onPostCreate之前被调用







                                                          
