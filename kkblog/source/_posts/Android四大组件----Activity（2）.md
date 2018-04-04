---
title: Activity(2)
date: 2017-09-07 17:19:20
tags: [activity,启动过程]
categories: Android
---
Android四大组件----Activity（2）
================================

-   [基础概念](#Android四大组件----Activity（2）-基础概念)

    -   [通信](#Android四大组件----Activity（2）-通信)

        -   [1.ActivityManagerService](#Android四大组件----Activity（2）-1.ActivityMa)

        -   [2.IApplicationThread](#Android四大组件----Activity（2）-2.IApplicati)

        -   [3.IActivityManager](#Android四大组件----Activity（2）-3.IActivityM)

        -   [4.基本联系](#Android四大组件----Activity（2）-4.基本联系)

    -   [协助](#Android四大组件----Activity（2）-协助)

        -   [1.ActivityRecord](#Android四大组件----Activity（2）-1.ActivityRe)

        -   [2.TaskRecord](#Android四大组件----Activity（2）-2.TaskRecord)

        -   [3.ActivityStack](#Android四大组件----Activity（2）-3.ActivitySt)

        -   [4.ActivityThread](#Android四大组件----Activity（2）-4.ActivityTh)

    -   [5.Instrumentation](#Android四大组件----Activity（2）-5.Instrument)

-   [基本过程](#Android四大组件----Activity（2）-基本过程)

-   [一些问题](#Android四大组件----Activity（2）-一些问题)

    -   [问题1](#Android四大组件----Activity（2）-问题1)

    -   [问题2](#Android四大组件----Activity（2）-问题2)

### 基础概念

#### 通信

##### 1.ActivityManagerService

主要管理应用进程的生命周期以及进程的Activity,Service,Broadcast等等。

可以分为client端与Server端：

-   Client端运行在app的各个进程，这些进程实现了具体的activity和service，通过调用系统接口来完成显示。

-   Service端运行在系统进程中，是系统级别的AMS的实现，响应client端的系统调用请求，并且管理client端的各个APP的生命周期。

##### 2.IApplicationThread

该接口定义了AMS访问App的接口。AMS通过这个接口的实现控制App的进程完成App的响应。

实现：ApplicationThread。实质上是一个Binder对象，在进程启动，创建ActivityThread时实例化，AMS通过它的代理ApplicationThreadProxy和Activity进行进程间通信。

##### 3.IActivityManager

该接口定义了App访问AMS的接口。App通过这个接口的实现对请求AMS完成某些操作。

实现：ActivityManagerNative。Activity通过代理ActivityManagerProxy将请求送到AMS进行进程间通信。

##### 4.基本联系

-   一个进程对应一个ActivityThread实例，这个进程里面所有的activity对应这一个ActivityThread实例

-   一个进程对应一个ApplicationThread对象，此对象是ActivityThread 与
    ActivityManagerService连接的桥梁。

 

#### 协助

##### 1.ActivityRecord

-   记录一个Activity的相关信息

##### 2.TaskRecord

-   记录当前Task中所有Activity：ArrayList\<ActivityRecord\> mActivities

-   ActivityStack成员stack记录Task所在的栈，用于执行ActivityStack的方法

##### 3.ActivityStack

管理协调ActivityRecord和TaskRecord

-   记录所有的栈

-   通知WindowManagerService的监听器

##### 4.ActivityThread

-   应用程序的入口

-   管理【应用程序的主线程】-----它自己不是线程

-   根据AMS的要求，进行Activity的调度

#### 5.Instrumentation

-   管理监测Android控件的运行

-   根据ActivityThread的要求，完成Activity的生命周期控制

 

### 基本过程

step1：无论是launcher启动，还是activity内部调用startActivity接口来启动新的Activity，都是通过Binder的进程间通信进入到AMS进程中，然后调用ActivityManagerService.startActivity。

期间主要经历以下过程：

     
 1)在Launcher.startActivity中由于.MainActivity在注册文件中设置了intent，所以点击图标时，launcher会将这个activity启动。
问题1：intent的信息怎么被Launcher找到？

     
 2)在Activity.startActivityForResult中，通过mMainThread（应用的主线程）.getApplicationThread获得了**ApplicationThread**对象，用于AMS和ActivityThread的通信，然后执行了Instrumentation.execStartActivity。

     
 3)在Instrumentation.execStartActivity中，通过ActivityManagerNative.getDefault返回ActivityManagerService的远程接口，得到了**ActivityManagerProxy**接口。

     
 4)通过**ActivityManagerProxy**接口，向AMS发出请求，也就是执行ActivityManagerService.startActivity。

step2：AMS.startActivity将操作转发给成员变量mMainStack（ActivityStack类型）的startActivityMayWait函数，来准备要启动的Activity的相关信息。

之后会经历以下过程：

     
 1)在 ActivityStack.startActivityLocked中，可以从传进来的参数中得到调用者的信息，即Laucher应用程序的进程信息。

     
 2)在ActivityStack.startActivityUncheckedLocked中，根据当前Activity的启动模式、taskAfiinity属性，设置的intent
flags等选择Task。为当前新创建的ActivityRecord找到Task。

       比较复杂，其实是理解启动模式的关键。还是需要结合源码才能理解。

     
 3)根据启动模式的相关知识，寻找Task还涉及到Task顶端的Activty。这个过程由resumeTopActivityLocked完成，主要就是看我们要启动的Activity是否在栈顶，如果不是的话，需要将当前运行的Activity暂停。这个工作还是由ActivityStack.startPausingLocked()来完成。

step3：ActivityStack在startPausingLocked中会将需要暂停的Activity暂停，主要经历以下过程：

     
 1)通过ApplicationThread.schedulePauseActivity进行通信，告诉ActivityThread执行handlePauseActivity

     
 2)ActivityThread接收到通知，于是通过ActivityManager向ActivityManagerService发起activityPaused的请求

     
 3) ActivityManagerService接收到请求，把工作交给ActivityStack，它进行了Activity的停止工作

     
 4)停止了该停止了Activity，又通过ApplicationThread来告诉系统服务进程要进行真正的Activity启动调度了。这个系统进程就是在step1中发出startActivity请求的Activity对应的进程，比如对于点击程序图标启动，那么就是Launcher所在进程，而对于Activity内部调用startActivity的情景，这个就是这个Activity所在进程。

step4：ApplicationThread不执行真正的启动操作，它通过ActivityManagerService.activityPause接口进入到AMS进程中，看是否需要创建新的进程来启动Activity。
如果是点击应用图标启动的话， ActivityManagerService
在这步会调用startProcessLocked来创建一个新的进程，而对于通过在Activty内部调用startActivity来启动新的Activity来说，这步不需要，因为新的activity就在原来的activity所在的进程中启动。

step5：ActivityManagerService调用ApplicationThread.scheduleLaunchActivity，通知相应的进程执行Activity的操作。
ApplicationThread把这个启动Activity的操作转发给ActivityThread，ActivityThread发送msg，接收到消息后进行handleLaunchActivity操作，该操作主要是调用进行PerformLaunchActivity

主要是以下过程：

        1）ActivityThread
调用Instrumentation的newOnActivity方法，通过ClassLoader导入相应的Activity类

       
2）创建Application对象（一个应用只有一个Application），然后通过attach把activity的上下文信息（Context）设置到mainActivity去。

       
3）Instrumentation调用callActivityOnCreate()执行Activity的onCreate()方法。

### 一些问题

#### 问题1

Launcher怎么获得intent信息？

-   涉及到PackageManagerService。它负责apk的安装、卸载等，在安装apk时会进行package的解析，把四大组件的相关信息存储下来。解析时，PMS中有相关的操作，对intentFiters查找、添加与匹配。对于一个新安装的应用，PMS找到intent-fileter中action为“android.intent.action.MAIN”并且category为“android.intent.category.LAUNCHER”的Activity，就会为这种应用程序创建桌面图标。

#### 问题2

ActivityThread如何管理主线程？

-   从上述对Activity启动过程的简单介绍中，可以看到ActivityThread会执行handlePauseActivity、PerformLaunchActivity等操作，也就是说ActivityThread控制了Activity的暂停、创建等操作，在ActivityThread的main()函数中可以看到它做了这样的事：

-   1)创建Looper

-   2)创建ActivityThread对象

-   3)创建binder对象ApplicationThread，将它和当前ActivityThread以及Application绑定

-   4)Looper循环，Handler接收消息，然后ActivityThread就可以执行handle方法，通过ActivityManager向ActivityManagerService发出请求

 

参考资料：

<http://blog.csdn.net/luoshengyang/article/details/6689748>

<http://blog.csdn.net/luoshengyang/article/details/6685853>

<http://www.2cto.com/kf/201610/554316.html>

 
