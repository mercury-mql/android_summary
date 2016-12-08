# Service

***

**Service并不是一个独立的进程，除非通过android：process等指明，否则它运行在APP的进程中。**

**Service并不是一个线程，它运行在APP的主线程中**


## 生命周期

![](images/service_lifecycle.png)

There are two reasons that a service can be run by the system. If someone calls Context.startService() then the system will retrieve the service (creating it and calling its onCreate() method if needed) and then call its onStartCommand(Intent, int, int) method with the arguments supplied by the client. The service will at this point continue running until Context.stopService() or stopSelf() is called. Note that multiple calls to Context.startService() do not nest (though they do result in multiple corresponding calls to onStartCommand()), so no matter how many times it is started a service will be stopped once Context.stopService() or stopSelf() is called; however, services can use their stopSelf(int) method to ensure the service is not stopped until started intents have been processed.

For started services, there are two additional major modes of operation they can decide to run in, depending on the value they return from onStartCommand(): START_STICKY is used for services that are explicitly started and stopped as needed, while START_NOT_STICKY or START_REDELIVER_INTENT are used for services that should only remain running while processing any commands sent to them. See the linked documentation for more detail on the semantics.

Clients can also use Context.bindService() to obtain a persistent connection to a service. This likewise creates the service if it is not already running (calling onCreate() while doing so), but does not call onStartCommand(). The client will receive the IBinder object that the service returns from its onBind(Intent) method, allowing the client to then make calls back to the service. The service will remain running as long as the connection is established (whether or not the client retains a reference on the service's IBinder). Usually the IBinder returned is for a complex interface that has been written in aidl.

A service can be both started and have connections bound to it. In such a case, the system will keep the service running as long as either it is started or there are one or more connections to it with the Context.BIND_AUTO_CREATE flag. Once neither of these situations hold, the service's onDestroy() method is called and the service is effectively terminated. All cleanup (stopping threads, unregistering receivers) should be complete upon returning from onDestroy().

## 分类

## Standard

## Bound

## 前台Service

## AIDL

## 更便捷的方式——IntentService

[http://android.xsoftlab.net/guide/components/services.html](http://android.xsoftlab.net/guide/components/services.html)

[http://android.xsoftlab.net/reference/android/app/Service.html](http://android.xsoftlab.net/reference/android/app/Service.html)