---
title: Activity(1)
date: 2017-09-06 14:09:30
tags: [activity,launchmode,intent flag,状态恢复与保存]
categories: Android
---

Android四大组件----Activity（1）
================================

-   [简介](#Android四大组件----Activity（1）-简介)

    -   [Activity的主要作用](#Android四大组件----Activity（1）-Activity的主要作)

-   [通信方式（待续）](#Android四大组件----Activity（1）-通信方式（待续）)

-   [关联处理](#Android四大组件----Activity（1）-关联处理)

    -   [关联起作用的两种情况](#Android四大组件----Activity（1）-关联起作用的两种情况)

        -   [启动Activity的Intent包含FLAG_ACTIVITY_NEW_TASK](#Android四大组件----Activity（1）-启动Activity的I)

        -   [Activity的allowTaskReparenting属性被设置为”true"](#Android四大组件----Activity（1）-Activity的all)

-   [加载方式](#Android四大组件----Activity（1）-加载方式)

    -   [使用清单文件](#Android四大组件----Activity（1）-使用清单文件)

        -   [singleTop](#Android四大组件----Activity（1）-singleTop)

        -   [singleTask](#Android四大组件----Activity（1）-singleTask)

        -   [singleInstance](#Android四大组件----Activity（1）-singleInstan)

        -   [singleTop和singleTask的区别](#Android四大组件----Activity（1）-singleTop和si)

    -   [使用intent Flag](#Android四大组件----Activity（1）-使用intentFlag)

        -   [FLAG_ACTIVITY_NEW_TASK](#Android四大组件----Activity（1）-FLAG_ACTIVIT)

        -   [FLAG_ACTIVITY_SINGLE_TOP](#Android四大组件----Activity（1）-FLAG_ACTIVIT)

        -   [FLAG_ACTIVITY_CLEAR_TOP](#Android四大组件----Activity（1）-FLAG_ACTIVIT)

        -   [SINGLE_TOP和CLEART_TOP的配合使用](#Android四大组件----Activity（1）-SINGLE_TOP和C)

            -   [为什么？](#Android四大组件----Activity（1）-为什么？)

            -   [可能出现的Bug](#Android四大组件----Activity（1）-可能出现的Bug)

        -   [NEW_TASK和CLEART_TOP的配合使用](#Android四大组件----Activity（1）-NEW_TASK和CLE)

-   [Lifecycle](#Android四大组件----Activity（1）-Lifecycle)

    -   [生命状态](#Android四大组件----Activity（1）-生命状态)

    -   [LifeTime](#Android四大组件----Activity（1）-LifeTime)

    -   [生命周期方法](#Android四大组件----Activity（1）-生命周期方法)

-   [状态保存与恢复](#Android四大组件----Activity（1）-状态保存与恢复)

    -   [onSaveInstanceState()](#Android四大组件----Activity（1）-onSaveInstan)

    -   [onRestoreInstanceState()或onCreate()](#Android四大组件----Activity（1）-onRestoreIns)

-   [参考资料](#Android四大组件----Activity（1）-参考资料)

### 简介

#### Activity的主要作用

-   显示UI

-   和用户进行交互，对用户的操作进行监听并响应

 

### 通信方式（待续）

1.Activity之间的通信

2.Activity和Service之间的通信

3.Activity和Fragment之间的通信

 

### 关联处理

关联代表Activity属于哪个Task，默认情况下，同一个应用中的所有Activities属于一个Task，但是可以通过<activity>元素的taskAffinity属性修改任何给定Activity的关联任务。

该属性值必须不同于<manifest>元素中默认的软件包名。

#### 关联起作用的两种情况

##### 启动Activity的Intent包含FLAG_ACTIVITY_NEW_TASK

-   默认情况下，存在此FLAG时，Activity通常会被放入新的Task中，但如果使用了taskAffinity将它与现有Task关联，Activity会在现有Task中启动，否则会开始新的Task（singleTask的实验证明了不会有新的Task，这里待定）。

-   如果使用此FLAG使得Activity开始新任务，当用户按“home"按钮离开时，必须为用户提供回到任务的方式。比如通知管理器所启动的Activity都是在别的Task中。

##### Activity的allowTaskReparenting属性被设置为”true"

-   Activity可以从它当前所在的Task移动到与其具有关联的任务。

-   比如一个天气预报Activity，它默认与旅行应用关联。如果不是这个应用的其他某个Activity启动了这个天气预报，之后旅行应用又启动了这个天气预报，那么天气预报Activity会被重新分配到旅行应用的Task。疑问：这样的意义何在？？

### 加载方式

当Activity A用设置intent flag的方式定义了Activity
B应当如何和任务关联，而Activity B又在清单中设置了launchMode，那么Activity A的设置优先级高于B的设置。也就是Intent flag的优先级大于launchMode

#### 使用清单文件

共有四种，主要是控制Activity的创建模式。

**standard**

-   如果intent来自同一个应用，且Activity没指定taskAffinity，那么生成Activity的新实例放在发出启动请求的Activity所在的Task顶端。

-   如果intent来自其他应用，那么会生成这个Activity的新实例，并以这个Activity为根创建一个新的Task

>   使用场景：

-   大多数撰写邮件或者发布社交网络状态的Activity。此时每个Activity实例服务于一个intent。比如从别的应用分享内容给微信朋友，跳到了那个选择联系人的list，这个页面很可能符合这种场景。
      
    可能出现的Bug：

-   使用ApplicationContext去启动standard模式的Activity会出现报错  
    因为standard模式的Activity会默认进入启动它的Activity所属的任务栈中，但是非Activity类型的Context没有所谓的任务栈。（why，因为任务栈是对于Activity来说的呀，而不是Application)  
    solution：使用FLAG_ACTIVITY_NEW_TASK启动Activity，启动的时候Activity就会进入新的Task。

##### singleTop

-   如果intent来自同一个应用，视栈顶是否有它的实例而定。如果已经有实例在栈顶了(发起启动的和已经启动的Task中），就不产生新实例，会通过
onNewIntent()把intent发送给现有的实例；如果有实例但不位于栈顶，就再创建新的实例并放入栈顶(实例可能重复）

-   如果intent来自其他应用，那么会生成这个Activity的新实例，并以这个Activity为根创建一个新的Task。  
      
    使用场景：

-   新闻类或者阅读类App的内容页面。在读新闻的时候，来了个该新闻APP的消息通知，点击这个通知就会打开新闻APP，由于此时这个App实例位于栈顶，那么此时就直接调用onNewintent()，无需产生新的实例

-   搜索功能的页面。在带有搜索功能的页面中，如果每次搜索都生成新的Activity，那么我们想回到最初搜索前的页面，就需要按很多次Back，但如果使用singleTop模式，每次搜索时，把intent发送给现有的Activity，更新搜索结果即可，只需要一次Back就可以回到搜索前的页面。

##### singleTask

-   如果intent来自同一个应用，如果栈中**存在这个Activity的实例**就会复用这个Activity，不管它是否位于栈顶，复用时，会将它上面的Activity全部出栈，并且将intent发送给该实例的onNewIntent()方法；如果当前栈没有这个实例，就新建一个，并且放在栈顶。（官方文档说是创建新Task并且以他为根，实验证明这种说法不完全正确）。

>   疑问：当前栈没有实例以及别的栈也没有实例，究竟是否一样结果？是新建后放栈顶，还是以实例为根创建Task？待实验证明。

>   实验：

>   1）同一个应用下，两个Stack，我们的应用在Stack\#1，其中已经有一个Task，**没有设置taskAffinity**，所有栈中都没有要找的Act。从.mainActivity跳转到另一个模式为singleTask的TestAct中，利用adb命令可以看到，Act新建后放到当前Task顶端。没有以实例为根创建新的Task。

>   2）同一个应用下，两个Stack，我们的应用在Stack\#1，其中已经有一个Task，**设置了TestAct的taskAffinity**，所有栈中都没有要找的Act。通过adb命令可以看到，在当前Stack中新建了一个Task，也就是以新建的Activity为根创建了Task。

>   猜想：

>   如果设置了taskAffinity，当前Task没有实例，就找它想要的Task，查找是否有与这个Activity相同taskAffinity的Task已经存在。如果不存在，创建实例时以这个Activity为根创建一个新的Task，这个新的实例就在这个Task中，以后每次调动都不会产生新的实例；如果存在Task，就再看是否有实例从而决定是复用还是新建。

>   但如果没有设置taskAffinity这个属性，它默认和当前应用处于一个Task，所以直接查找发出intent的当前Task，如果找不到就直接在当前Task顶端新建实例。

>   注意的是，**虽然系统创建了一个新的Task，但是只要按下返回键还是会回到原来的Activity**。

-   如果intent来自别的应用，无论哪个Task存在这个Activity的实例都会复用这个Activity，不管它是否位于栈顶，复用时，会将它上面的Activity全部出栈，并且将intent发送给该实例的onNewIntent()方法；如果系统中（当前栈和别的Task）都没有这个Activity的实例，那么就创建新实例并以它为根创建新的Task。

      使用场景：

-   浏览器的主界面。不管从多少个应用启动浏览器，只会启动主界面一次，其余情况都会走onNewIntent()，同时会清空主界面上面的其他页面。

-   邮件客户端的收件箱。不管从哪个应用进入收件箱，都会结束客户端中别的页面，进入到收件箱页面。

##### singleInstance

-   和singleTask类似，不但要创建一个新的Task，而且该Task中只能有这个实例。其效果相当于多个应用共享一个应用，不管谁激活该Activity 都会进入同一个应用中。

      使用场景：

-   几乎不使用这种模式，除非是100%确定这个应用只有一个Activity。

-   闹铃提醒，将闹铃提醒与闹铃设置分离。此时要保证这个应用只有闹铃提醒这个Act。

##### singleTop和singleTask的区别

当已经存在要启动的Activity时

-   如果它在栈顶，那么都会把intent发送给onNewIntent()。

-   如果它不在栈顶，singleTop会在栈顶新建实例，而singleTask会把这个Activity上面的实例弹出，再发送intent给栈顶的这个Activity。  
    ***也就是说，singleTop会造成重复的实例***。

#### 使用intent Flag

##### FLAG_ACTIVITY_NEW_TASK

与singleTask有相同行为。

##### FLAG_ACTIVITY_SINGLE_TOP

与singleTop有相同行为。

##### FLAG_ACTIVITY_CLEAR_TOP

如果已经有了Activity实例，那么就会销毁Task中当前Activity之上的所有Activity，然后把intent发送给已经位于Task顶端的Act实例的onNewIntent()。

##### SINGLE_TOP和CLEART_TOP的配合使用

###### 为什么？

      如果要启动的Activity为”standard"：

-   只使用CLEAR_TOP，想要启动的Activity已存在时，该实例**以及**它上面的Activity会被销毁，然后创建新实例(onCreate())

-   配合SINGLE_UP使用**或者**要启动的Activity为**“standard"之外**的模式，想要启动的Activity已存在时，该实例不会被销毁，只有它上面的实例会被销毁，然后调用onNewIntent()

-   详情：

<https://stackoverflow.com/questions/31926315/why-do-people-like-to-pair-clear-top-and-single-top-in-android>
<https://developer.android.com/reference/android/content/Intent.html#FLAG_ACTIVITY_CLEAR_TOP>

###### 可能出现的Bug

问题：<https://stackoverflow.com/questions/5489592/single-top-clear-top-seem-to-work-95-of-the-time-why-the-5>

有人回答道这涉及到并发问题，如果CLEAR_UP和SINGLE_TOP被同时读取，CLEAR_UP会调用onCreate()，SINGLE_UPC此时也没有找到Act，也调用onCreate()，造成一个Activity同时创建两次。

存疑：逻辑或 \| ，运算优先级为从左到右，会发生同时读取两个条件的情况吗？

猜想：此时intent进行setFlags，不是简单的立刻进行false/true的判断，而是需要使用这两个参数，所以可能在启动过程中，对参数进行读取，然后根据参数进行相应的操作。结合对启动过程的了解，答案应该还是在ActivityStack.startActivityUncheckedLocked中。

 

##### NEW_TASK和CLEART_TOP的配合使用

-   如果在一个Task的根Act中使用这个FLAGS,可以找到当前Task中运行的Activity实例并使它在前台运行，并且清除这个根Act和需要的Activity之外的其余实例。

 

### Lifecycle

#### 生命状态

-   活动（running）状态：当Activity处于可见，位于屏幕前台，获得焦点，正在和用户进行交互时为活动状态，此时它位于任务栈的最顶端。

-   暂停状态：Activity半可见或者透明时（比如出现了toast、alertdialog等时），它没有焦点，无法和用户进行交互，此时活动仍是存活的，它保留着所有的状态和成员信息。系统内存资源不够时可能被杀死。

-   停止状态：完全不可见时为停止状态，但仍保持所有的状态和成员信息。随时可能会被系统杀死。

-   非活动状态：Activity处于暂停或停止状态，Activity被手动终止，或者被系统回收资源时,可以称为非活动状态。

#### LifeTime

 

-   entiere lifetime：调用onCreate()到相应的调用onDestroy()

 

-   visible lifetime: 调用onStart()到相应的调用onStop()

 

-   foreground lifetime：调用onResume()到相应的调用onPause()

#### **生命周期方法**

**      以下六个方法从外向内两两对应：**

-   onCreate（）：活动第一次启动时会首先调用该方法

-   onStart（）：有一个参数，可能为null， 或者是onSaveInstanceState()
    保存的状态信息。
    启动后或者当活动从停止状态转到活动状态时会调用该方法。onStart（）是activity界面被显示出来的时候执行的，用户可见，包括有一个activity在他上面，但没有将它完全覆盖，用户可以看到部分activity但不能与它交互。

-   onResume（）：活动和用户进行交互时会调用该方法，当从pause状态转为活动状态时回调该方法，是当该activity与用户能进行交互时被执行，用户可以获得activity的焦点，能够与用户交互。

-   onPause（）：一个活动在前台运行时，别的活动需要前台运行，这时该活动会保存当前状态信息，活动变为暂停状态

-   onStop（）：活动不需要展示给用户时触发该方法，活动变为停止状态

-   onDestroy（）：活动停止时调用该方法

 

-   onRestart（）：活动从停止状态转为活动状态时调用该方法

onStart()和onResume()区别：

-   onStart()通常就是onStop()（也就是用户按下了home键，activity变为后台后），之后用户再切换回这个Activity就会调用onRestart()而后调用onStart()

-   onResume()是onPause()（通常是当前的Activity被暂停了，比如被另一个透明或者Dialog样式的Activity覆盖了），之后dialog取消，activity回到可交互状态，调用onResume()。

### 状态保存与恢复

Activity遇到意外被关闭时，需要对一些状态如文本输入、ListView位置进行记录

#### onSaveInstanceState()

     1）在回调onStop()之前调用

     2）与onPause()的调用顺序不一定

  3）只有当Activity可能会重建时，比如系统资源不足或者配置改变导致Activity关闭时，才会调用该函数


#### onRestoreInstanceState()或onCreate()

     1）onRestoreInstanceState()在回调onStart()之后调用，此时Bundle一定不为null

     2）回调onCreate()时Bundle可能为null，所以需要进行判断

   
 3）根据两者差异，如果需要根据Activity是第一次创建还是重建进行不同的初始化操作，可以使用onCreate()

   
 4）官方文档建议使用onRestoreInstanceState()，有利于继承这个Activity的子类决定是否使用父类的状态恢复实现

### 参考资料

<https://inthecheesefactory.com/blog/understand-android-activity-launchmode/en>

<https://developer.android.com/reference/android/app/Activity.html>

<https://developer.android.com/guide/components/tasks-and-back-stack.html?hl=zh-cn>

<https://stackoverflow.com/questions/4342761/how-do-you-use-intent-flag-activity-clear-top-to-clear-the-activity-stack>

<https://stackoverflow.com/questions/12683779/are-oncreate-and-onrestoreinstancestate-mutually-exclusive>

<https://developer.android.com/reference/android/app/Activity.html#onSaveInstanceState(android.os.Bundle)>

<https://developer.android.com/reference/android/app/Activity.html#onRestoreInstanceState(android.os.Bundle)>

<https://www.reddit.com/r/androiddev/comments/2afx13/onrestoreinstancestate_vs_oncreate/>

 
