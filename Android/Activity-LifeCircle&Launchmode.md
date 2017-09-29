# Activity生命周期

![activity_lifecycle](activity_lifecycle.png)



## 有关生命周期的几个回调方法

- ### onCreate()

系统第一次创建该Activity实例时会回调该方法。该方法一般承担着一些初始化工作，比如调用setContentView去加载界面布局，初始化Activity所需的数据并绑定到对应的View上。

- ### onRestart()

Activity从onStop状态，将要重新进入前台（由不可见到可见）。

- ### onStart()

Activity正在启动，已经时可见的了，但还没到前台，无法和用户交互。

- ### onResume()

Activity已经在前台，可以与用户交互。

- ### onPause()

Activity离开前台，不能与用户交互。这也是Activity接收到用户要离开该Activity指令时所回调的第一个方法。比如：

1. 有电话打进来
2. 手机息屏
3. 进入其他的Activity
4. 打开一个新的半透明的Activity
5. 在Android 7.0之后的multi-window模式下，失去焦点的Activity会调用onPause

onPause适合做轻量级的回收工作，比如停止动画等，而不适合进行太耗时的操作，因为旧的onPause执行完后，新的Activity的onResume才会执行。

执行完onPause后并不意味着activity就离开了Paused 状态了，因为此时旧的activity仍然可见且新的activity还未进入resume状态。如果activity变得完全不可见了，回调onStop。

- ### onStop()

Activity即将停止，不再可见。

onStop适合做稍微重量级的回收工作，比如释放导致内存泄漏的资源，保存数据到数据库中。

有时候由于高优先级的应用需要内存，系统不足以分配，就把拥有该Activity的进程杀死来释放内存，这样就不会调用onDestory。(It is possible for the system to kill the process hosting your activity without calling the activity's final `onDestroy()` callback.)

- ### onDestroy()

Activity即将被销毁，Activity生命周期中最后一个回调，适合做最终的资源释放与回收工作。

有两种情况下，系统回调此方法。一是Activity的finish()被调用，二是系统为了节约内存而把activity所在的进程杀死(the system is temporarily destroying the process containing the activity to save space)。这两种方式可以用 `isFinishing()` 来区分，前者 `isFinishing()` 返回true。



**小结**

onCreate与onDestroy是一对回调，标志一个Activity的诞生与消亡；

onStart与onStop是一对回调，标志一个Activity是否可见；

onResume与onPause是一对回调，标志一个Activity是否能与用户交互，是位于前台还是后台。



## 典型场景

- 由ActivityA跳转到ActivityB

回调如下：

-> A的onPause 

-> B的onCreate,onStart,onResume

-> 如果A不可见则还会调用A的onStop



- Android7.0的Muti-Window分屏模式

当Activity获得焦点，回调onResume；

当Activity失去焦点，回调onPause，但由于此时Activity仍处于可见状态，所以不会回调onStop

- 点击home键并重回Activity

-> onPause-> onSaveInstanceState -> onStop -> onRestart -> onStart -> onResume

- 异常情况(如Activity被杀死，Activity由竖屏变为横屏)

当系统配置发生改变后，Activity会被销毁再重建，系统会在onStop之前调用onSaveInstanceState来保存当前Activity的状态，与onPause没有既定的时序关系。onSaveInstanceState只是为将来Activity可能被重建保存快照，如果Activity果真被重建了，系统把onSaveInstanceState所保存的Bundle对象作为参数传递给onCreate与onRestoreInstanceState，onRestoreInstanceState在onStart调用之后。

如果想让系统配置发生改变后，不想系统重新创建Activity可以给Activity指定configChanges属性。比如不想让Activity在屏幕旋转的时候重新创建可以给Activity添加属性：`android:configChanges="orientation"`。

- Activity在前台，设置了启动模式为singleTop 或者 singleTask，再次启动

-> onPause -> onNewIntent -> onResume

这是我发现唯二能够由onPause到onResume的情况。



# Activity的启动模式

1. Standard：标准模式，每次启动一个Activity都会重新创建一个新的实例，不管这个实例是否已经存在。Standard模式的Activity默认会进入启动它的Activity所属的任务栈中。
2. SingleTop：栈顶复用模式。如果新的Activity已经位于任务栈的栈顶，此Activity不会被重新创建，同时它的onNewIntent方法会被回调。
3. singleTask：栈内复用模式。只要栈内存在该Activity，那么此Activity不会重新创建实例，和singleTop一样，系统也会回调其onNewIntent方法。
4. singleInstance：单例模式。每次启动Activity都需要新开辟一个栈，并且此栈不能有第二个Activity。

## TaskAffinity与singleTask

Task其实是Activity集合的概念。Affinity意为亲和力，标识了一个Activity任务栈的名字，默认所有的Activity所需的任务栈名字为应用的包名。

TaskAffinity属性要和singleTask启动模式或allowTaskRepresenting属性配对使用，在其他情况下没有意义。

- 如果待启动的Activity是以singleTask模式来启动的，它会运行在名字和自身TaskAffinity相同的任务栈中
- 如果应用A启动了应用B的某个Activity，这个Activity的allowTaskRepresenting属性为true，那么当应用B被启动后，此Activity会直接从应用A的任务栈转移到应用B的任务栈中。

当我们用ApplicationContext去启动standard模式的Activity的时候会报错。因为standard模式的Activity默认会进入启动它的Activity所属的任务栈中，但非Activity类型的Context没有所谓的任务栈。

## IntentFilter的匹配规则

启动Activity分为两种，显式调用和隐式调用。显式调用需要明确地指定被启动对象的组件信息，包括包名和类名；而隐式启动需要明确指定组件的信息。

隐式启动需要同时匹配intent-filter中的action，category和data信息。一个intent-filter中的action，category和data可以有多个，所有的action，category和data分别构成不同类别。

### action

action是一个字符串。一个过滤规则可以有多个action，那么Intent中的action必须存在且和过滤规则中的其中一个action相同才算匹配。

### category

category是要给字符串，要求Intent中可以没有category，但如果存在，那么每个category都必须与过滤规则中的任何一个category相同。

### data

data由两部份组成，mimeType和URI。data匹配规则与action类似。