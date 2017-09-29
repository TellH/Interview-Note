## Java访问C

当调用native函数时，Java会自动产生一个对应的C中函数名称。例如Framework中AssetManager类中声明了以下方法：

```java
private native final void init();
```

该方法对应在C中是：

```c
static void android_content_AssetManager_init(JNIEnv* env, jobject clazz)
```

当Java调用native时，编译器会向native引擎传递调用者的包名，以及函数名和参数类型。

在产生的C函数中，会包含至少两个参数。前者是JNIEnv对象，该对象是一个JVM所运行的环境，相当于JVM的管家，通过它可以访问JVM内部的各种对象；第二个参数jobject是调用该函数的对象。



## C访问Java

C调用Java时，也需要把想要访问的类名，函数名和参数传递给Java引擎。其步骤如下：

1. 获取Java对象的类

```c
cls = env->GetObjectClass(jobject)
```

2. 获取Java方法的id

```c
jmethodId mid = env->GetMethodId(cls, "method_name", "(Ljava/lang/String;)V");
```

第三个参数是函数签名，包括参数和返回值。

3. 找到这个函数，并调用它

```c
env->CallXXXMethod(jobject,mid,ret);
```

XXX代表函数的返回值类型，包括Void，Object，Boolean，Byte，Char，Short，Int，Long，Float和Double。