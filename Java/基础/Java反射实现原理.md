## 从一段示例代码开始

```java
        Class clz = Class.forName("ClassA");
        Object instance = clz.newInstance();
        Method method = clz.getMethod("myMethod", String.class);
        method.invoke(instance, "abc","efg");
```

前两行实现了类的装载、链接和初始化（newInstance方法实际上也是使用反射调用了`<init>`方法），后两行实现了从class对象中获取到method对象然后执行反射调用。试想一下，如果`Method.invoke`方法内，动态拼接成如下代码，转化成JVM能运行的字节码，就可以实现反射调用了。

```java
     public Object invoke(Object obj, Object[] param){
        MyClass instance=(MyClass)obj;
        return instance.myMethod(param[0],param[1],...);
     }
```

## Class和Method对象

Class对象里维护着该类的所有Method，Field，Constructor的cache，这份cache也可以被称作根对象。每次getMethod获取到的Method对象都持有对根对象的引用，因为一些重量级的Method的成员变量（主要是MethodAccessor），我们不希望每次创建Method对象都要重新初始化，于是所有代表同一个方法的Method对象都共享着根对象的MethodAccessor，每一次创建都会调用根对象的copy方法复制一份：

```java
    Method copy() { 
        Method res = new Method(clazz, name, parameterTypes, returnType,
                                exceptionTypes, modifiers, slot, signature,
                                annotations, parameterAnnotations, annotationDefault);
        res.root = this;
        res.methodAccessor = methodAccessor;
        return res;
    }
```



## 反射调用

```java
    public Object invoke(Object obj, Object... args)
        throws IllegalAccessException, IllegalArgumentException,
           InvocationTargetException
    {
        if (!override) {
            if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
                Class<?> caller = Reflection.getCallerClass();
                checkAccess(caller, clazz, obj, modifiers);
            }
        }
        MethodAccessor ma = methodAccessor;             // read volatile
        if (ma == null) {
            ma = acquireMethodAccessor();
        }
        return ma.invoke(obj, args);
    }
```

调用Method.invoke之后，先进行访问权限检查，再获取MethodAccessor对象，并调用MethodAccessor.invoke方法。MethodAccessor被同名Method对象所共享，由ReflectionFactory创建。创建机制采用了一种名为inflation的方式（JDK1.4之后）：如果该方法的累计调用次数<=15，会创建出NativeMethodAccessorImpl，它的实现就是直接调用native方法实现反射；如果该方法的累计调用次数>15，会创建出由字节码组装而成的MethodAccessorImpl。（是否采用inflation和15这个数字都可以在jvm参数中调整）

那么以示例的反射调用`ClassA.myMethod(String,String)`为例，生成MethodAccessorImpl类的字节码对应成Java代码如下：

```java
public class GeneratedMethodAccessor1 extends MethodAccessorImpl {    
    public Object invoke(Object obj, Object[] args)  throws Exception {
        try {
            MyClass target = (ClassA) obj;
            String arg0 = (String) args[0];
            String arg1 = (String) args[1];
            target.myMethod(arg0,arg1);
        } catch (Throwable t) {
            throw new InvocationTargetException(t);
        }
    }
}
```

## 性能

通过JNI调用native方法初始化更快，但对优化有阻碍作用。随着调用次数的增多，使用拼装出的字节码可以直接以Java调用的方式来实现反射，发挥了JIT的优化作用。

那么为什么Java反射调用被普通的方法调用慢很多呢？我认为主要有以下三点原因：

1. 因为接口的通用性，Java的invoke方法是传object和object[]数组的。基本类型参数需要装箱和拆箱，产生大量额外的对象和内存开销，频繁促发GC。
2. 编译器难以对动态调用的代码提前做优化，比如方法内联。
3. 反射需要按名检索类和方法，有一定的时间开销。



参考：

- http://www.fanyilun.me/2015/10/29/Java%E5%8F%8D%E5%B0%84%E5%8E%9F%E7%90%86/