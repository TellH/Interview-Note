【整理成博客】
http://blog.csdn.net/tellh/article/details/71308245

【草稿】

## 泛型的作用

泛型实现参数化类型的概念，使代码可以应用于多种类型，解耦类或方法与所使用的类型之间的约束。

## 泛型不支持协变

```java
class Fruit{}
class Apple extends Fruit{}
Fruit[] fruit = new Apple[10]; // OK
ArrayList<Fruit> flist = new ArrayList<Apple>(); // Not OK!
ArrayList<? extends Fruit> flist = new ArrayList<Apple>();// 使用通配符解决协变问题
```

## 上界通配符

```java
        List<? extends Fruit> flist = Arrays.asList(new Apple());
        Apple a = (Apple)flist.get(0); // No warning
        flist.contains(new Apple()); // Argument is ‘Object’
        flist.indexOf(new Apple()); // Argument is ‘Object’
        //flist.add(new Apple());   无法编译
```

通配符 `List<? extends Fruit>` 表示某种特定类型 ( `Fruit` 或者其子类 ) 的 List，但是并不关心（不知道）这个实际的具体类型到底是什么。注意，并不意味着这个List持有Fruit的任意类型。

由于List的具体类型并不确定，因此带有泛型类型参数的方法都无法正常调用。比如`add(T item);`，即使是传Object也无法通过编译。

但返回类型就是Fruit，与上界类型一样。

## 下界通配符

```java
    static void add(List<? super Apple> list) {
//        list.add(new Fruit()); // 无法编译
        Object object = list.get(0);// pass
    }
```
如何向泛型类型中 “写入” ( 传递对象给方法参数) 。
```java
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (int i = 0; i < src.size(); i++)
        dest.add(i, src.get(i));
}
```



## 无界通配符

`List<?> list` 表示 `list` 是持有某种特定类型的 List，但是不知道具体是哪种类型。而单独的 `List list` ，也就是没有传入泛型参数，表示这个 list 持有的元素的类型是 `Object`。

## getGenericSuperclass

```java
public class SubClass extends Base<String> { }
```

对SubClass.class调用`getGenericSuperclass`可以获取到T所绑定的类型。

```java
        Type type = SubClass.class.getGenericSuperclass();
        Type targ = ((ParameterizedType) type).getActualTypeArguments()[0];
        System.out.println(type); // SubClass<java.lang.String>
        System.out.println(targ); // class java.lang.String
```

具体用法可以参考Gson和Guice的源码：

https://github.com/google/guice/blob/abc78c361d9018da211690b673accb580a52abf2/core/src/com/google/inject/TypeLiteral.java#L94

https://github.com/google/gson/blob/master/gson/src/main/java/com/google/gson/internal/%24Gson%24Types.java

## 桥方法

为了使Java的泛型方法生成的字节码与1.5以前的字节码相兼容，由编译期自己生成的方法。顾名思义，桥方法是一座桥，沟通着泛型与多态。

可以通过`Method.isBridge()`方法来判断一个方法是否是桥接方法，在字节码中桥接方法会被标记为`ACC_BRIDGE`和`ACC_SYNTHETIC`。

```java
public class Fruit<T> {
    T value;
    public T getValue() {
        return value;
    }
}
public class Apple extends Fruit<String> {
    @Override
    public String getValue() {
        return "foo was call";
    }
}
```

反编译生成的字节码：

```
public class Apple extends Fruit<java.lang.String> {
  public Apple();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method Fruit."<init>":()V
       4: return

  public java.lang.String getValue();
    Code:
       0: ldc           #2                  // String calling
       2: areturn

  public java.lang.Object getValue();
    Code:
       0: aload_0
       1: invokevirtual #3                  // Method getValue:()Ljava/lang/String;
       4: areturn
}
```

编译器为我们自动生成了有一个桥方法，这个桥方法返回类型为Object，内部调用了我们自定义的另一个getValue方法。

在Java代码中，方法的特征签名只包括方法名称，参数顺序和参数类型，而字节码中的特征签名还包括方法返回值和受查异常表。因此，桥方法`public Object getValue()`与`public String getValue()`是可以被JVM区分而在同一个Class文件中共存的。

由于编译期泛型擦除机制，在父类中带泛型参数的方法会被替换成Object类型。要让子类重写父类带泛型参数的方法，需要通过桥方法直接复写父类的方法，然后桥方法再调用子类自定义的方法，就以上面作为例子，子类Apple中的桥方法`public Object getValue()`直接override父类Fruit的`public Object getValue()`，然后桥方法内部再调用子类Apple的`public String getValue()`。因此，Java利用桥方法在保证多态机制不被破坏情况下实现了泛型。

## 所有信息都被擦除了吗

所谓的擦除，仅仅是对方法的Code属性中的字节码（也就是方法内的逻辑代码）进行擦除，实际上元数据（类和接口的声明，类字段的声明）中还是保留了泛型信息。

```java
public class GenericClass<T> {                // 1  
    private List<T> list;                     // 2  
    private Map<String, T> map;               // 3  
    public <U> U genericMethod(Map<T, U> m) { // 4  
        List<String> list = new ArrayList<>(); // 5
        return null;
    }
}  
```

位于声明一侧的，源码里写了什么到运行时就能看到什么； 
位于使用一侧的，源码里写什么到运行时都没了。 

上面的代码中，1-4的T和U是保留在Class文件当中的，源码是什么，那么通过反射获取得到的就是什么。也就是说，在运行时，是无法获取到具体的T和U是什么类型的。

但运行时，在方法内部的局部变量的泛型信息是被全部擦除的。

**参考**:

- https://segmentfault.com/a/1190000005337789
- http://rednaxelafx.iteye.com/blog/586212
- http://blog.csdn.net/mhmyqn/article/details/47342577#
- http://blog.csdn.net/lonelyroamer/article/details/7868820