---
layout: post
title: 绕到 Kotlin 语法糖背后
categories: [Java, Kotlin]
description: Kotlin 语法糖背后是 Kotlin 编译器默默努力的结果，语法糖并不改变代码的功能和底层机制。
keywords: Kotlin
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---
<img src="{{site.url}}/images/posts/2025-01-09-Kotlin-Sugar/p1.png"/>
## 前言
在 Kotlin 语言设计哲学中有一个非常突出的特性：简洁。Kotlin 语言简洁，意味着 Kotlin 代码易于阅读和编写。易于阅读，不用在 RTFS 时皱着眉头。易于编写，不用在键盘上敲得手酸。

Kotlin 语言简洁的背后是 Kotlin 语法糖，语法糖在保证代码易读和易写的同时，不改变代码的功能和底层机制。

在项目开发过程中，语法糖用起来非常的爽，但其背后隐藏的实现往往容易被忽视。本文将围绕几种常见的 Kotlin 语法糖，探究其底层实现原理。

## 语法糖

### 中缀调用

#### 定义
中缀调用（infix notation）是一种特殊的函数调用。被调用的函数使用 `infix` 关键字修饰。定义中缀函数的条件是：
1. 必须是成员函数或扩展函数。
2. 函数只能有一个参数，参数不能是可变参数，且不能有默认值。
3. 函数使用 `infix` 关键字修饰。

#### 本质
中缀函数可以是成员函数或扩展函数：
```kotlin
// 成员函数
class MyClass {
    infix fun myInfixFunction(param: Type): ReturnType {
        // 函数体
    }
}

// 扩展函数
infix fun Type.myInfixFunction(param: AnotherType): ReturnType {
    // 函数体
}
```
下面的代码：
```kotlin
fun main() {
    val pair = 1 to "one"
}
```
编译后会生成类似的代码：
```java
public final void main() {
   Pair pair = TuplesKt.to(1, "one");
}
```
#### 示例

标准库中的 `to` 函数，用于创建 `Pair` 对象：

```kotlin
public infix fun <A, B> A.to(that: B): Pair<A, B> = Pair(this, that)

val pair = "key" to "value"

val map = mapOf(1 to "one", 2 to "two)
```

自定义中缀函数：
```kotlin
infix fun Int.pow(exponent: Int): Int = Math.pow(this.toDouble(), exponent.toDouble()).toInt()

val result = 2 pow 3 // 计算 2 的 3 次方
```
### 解构声明

#### 定义

解构声明允许将对象的多个属性解构为单独的变量。也就是允许展开单个复合值，并使用它来初始化多个单独的变量。


```kotlin
val p = Point(10, 20)
val (x, y) = p
此时：x = 10, y = 10
```
#### 本质
在解构声明中初始化的每个变量，都是调用名为 `componentN` 的函数，其中 `N` 是声明中变量的位置。

`componentN()` 函数：它是编译器自动生成的、用于获取对象属性的函数。
- 数据类（`data class`）：会自动为其主构造函数参数生成 `component1()`、`component2()` 等函数。
- 普通类：可以通过实现 `componentN()` 函数来自定义解构支持。

定义数据类 Point：
```kotlin
data class Point(val x: Int, val y: Int)
```
编译后会生成类似的代码：

```java
public final class Point {
   private final int x;
   private final int y;

   public final int getX() {
      return this.x;
   }

   public final int getY() {
      return this.y;
   }

   public Point(int x, int y) {
      this.x = x;
      this.y = y;
   }

   public final int component1() {
      return this.x;
   }

   public final int component2() {
      return this.y;
   }
}
```

使用`val (x, y) = Point(10, 20) `实际执行的代码是：

```kotlin
val x = point.component1() // 调用 component1() 获取 x 属性 
val y = point.component2() // 调用 component2() 获取 y 属性
```
#### 示例
从一个函数返回多个值：

```kotlin
fun getCoordinates(): Pair<Int, Int> {
    return Pair(10, 20) 
}

fun main() { 
    val (x, y) = getCoordinates() // 解构 Pair 类型 
    println("x: $x, y: $y") // 输出：x: 10, y: 20 
}
```
遍历集合：
```kotlin
fun main() {
    val map = mapOf("A" to 1, "B" to 2, "C" to 3)

    for ((key, value) in map) {  // 自动解构 Map.Entry
        println("$key -> $value")
    }
    // 输出：
    // A -> 1
    // B -> 2
    // C -> 3
}
```
Lambda 表达式：

```kotlin
val map = mapOf("A" to 1, "B" to 2)

map.forEach { (key, value) ->  // 解构 Lambda 参数
    println("$key -> $value")
}
```

自定义类支持解构：

```kotlin
class Point(val x: Int, val y: Int) {
    operator fun component1() = x
    operator fun component2() = y
}

fun main() {
    val point = Point(10, 20)
    val (x, y) = point  // 调用自定义的 component1() 和 component2()
    println("x: $x, y: $y")  // 输出：x: 10, y: 20
}

```
### 类委托

#### 定义

类委托通过 `by` 关键字将接口的实现委托给另一个对象。

```kotlin
class DelegatedClass(delegate: SomeInterface) : SomeInterface by delegate
```
`DelegatedClass` 实现了 `SomeInterface` 接口，但不需要手动实现接口的方法。 通过 `by` 将接口的实现委托给对象 `delegate`。

#### 本质

编译器会为委托类生成接口方法实现，在方法实现中，通过调用委托对象的方法来完成接口逻辑。

定义类委托：

```kotlin
interface SomeInterface {
    fun doSomething()
}

class DelegatedClass(val delegate: SomeInterface) : SomeInterface by delegate

```
编译后会生成类似的代码：
```java
public final class DelegatedClass implements SomeInterface {
   @NotNull
   private final SomeInterface delegate;

   @NotNull
   public final SomeInterface getDelegate() {
      return this.delegate;
   }

   public DelegatedClass(@NotNull SomeInterface delegate) {
      Intrinsics.checkNotNullParameter(delegate, "delegate");
      super();
      this.delegate = delegate;
   }

   public void doSomething() {
      this.delegate.doSomething();
   }
}
```
#### 示例

集合使用类委托：
```kotlin
class CountingSet<T>(val innerSet: MutableCollection<T>) : MutableCollection<T> by innerSet {
    var objectsAdded = 0

    override fun add(element: T): Boolean {
        objectsAdded++;
        return innerSet.add(element);
    }

    override fun addAll(elements: Collection<T>): Boolean {
        objectsAdded = elements.size;
        return innerSet.addAll(elements);
    }
}

val cset = CountingSet<Int>()
cset.addAll(listOf(1,2,3))

println("objectsAdded = ${cset.objectsAdded}")
objectsAdded = 3

println("size = ${cset.size}")
size = 2
```

### 伴生对象

#### 定义
伴生对象（Companion Object）是类中声明的一个特殊对象，使用 `companion`和`object` 关键字修饰。 

Kotlin 中没有 `static` 关键字，不能直接定义静态方法和静态属性，但使用伴生对象可以达到相同的效果。

伴生对象是一个与类绑定的单例对象，可以访问类的私有成员，通常用来提供工厂方法、静态方法或与类相关的常量：

-   伴生对象类似于 Java 中的 **static** 成员。
-   每个类最多只能有一个伴生对象。
-   伴生对象本质上是一个普通对象，只不过它与类绑定，并且可以通过类名直接访问。


```kotlin
class A {
    companion object{
        fun bar(){
            
        }
    }
}
//调用伴生对象方法
A.bar()
```

#### 本质

伴生对象是一个类的静态属性。

上面类A 编译后会生成类似的代码：
```java
public final class A {
   @NotNull
   public static final Companion Companion = new Companion((DefaultConstructorMarker)null);

   public static final class Companion {
      private Companion() {
      }

      public final void bar() {
      }

      public Companion(DefaultConstructorMarker $constructor_marker) {
         this();
      }
   }
}
```
Companion 对象是一个单例对象。

#### 示例

实现工厂模式并访问类的私有成员：

```kotlin
class User(private val id: Int, private val name: String) {
    companion object {
        fun create(id: Int, name: String): User {
            return User(id, name) // 伴生对象直接访问 User 的私有构造函数
        }
    }

    override fun toString() = "User(id=$id, name=$name)"
}

fun main() {
    val user = User.create(1, "Alice") // 使用伴生对象的工厂方法创建实例
    println(user)
}
```

### lambda 表达式在作用域中访问变量

#### 定义

lambda 表达式可以访问其所在作用域中的变量。这种访问外部作用域变量的能力被称为**闭包（closure）** 。

lambda 表达式可以访问：局部变量，外部类的成员变量，全局变量。

```kotlin
var sum = 0 
ints.filter { it > 0 }.forEach { sum += it } 
print(sum)
```
#### 本质
lambda 表达式可以捕获其作用域中的变量，并将这些变量与 lambda 绑定在一起，然后形成一个闭包对象。

##### 捕获不可变变量（val）

如果 lambda 表达式捕获的是不可变变量，Kotlin 编译器会直接将变量的值存储在闭包对象中。

捕获局部变量：
```kotlin
fun count() {
    //定义一个不可变局部变量
    val counter = 0
    val increment: () -> Unit = {
        //捕获不可变变量
        val c = counter
    }
}
```
上面代码编译后会生成类似的代码：
```java
public final void count() {
   final int counter = 0;
   Function0 increment = (Function0)(new Function0() {
      public Object invoke() {
         this.invoke();
         return Unit.INSTANCE;
      }

      public final void invoke() {
         int var1 = counter;
      }
   });
}
```
捕获全局变量：

```kotlin
class Counter {
    //定义一个不可变全局变量
    private val counter = 0

    fun count(){
        val increment: () -> Unit = {
            //捕获不可变变量
            val c = counter
        }
    }
}
```
上面代码编译后会生成类似的代码：
```java
public final class Counter {
   private final int counter;

   public final void count() {
      Function0 increment = (Function0)(new Function0() {
         public Object invoke() {
            this.invoke();
            return Unit.INSTANCE;
         }

         public final void invoke() {
            int var1 = Counter.this.counter;
         }
      });
   }
}
```
在 increment 对象中持有外部类 Counter 对象：`int var1 = Counter.this.counter;`。

##### 捕获可变变量（var）

如果 lambda 表达式捕获的是可变变量，Kotlin 编译器会将变量封装为一个对象（如 `Ref` 对象），通过间接引用来保证变量的修改对 lambda 和外部作用域都可见。在编译器生成的代码中，lambda 表达式会操作这个封装对象而非直接访问变量。

捕获局部变量：
```kotlin
fun count() {
    // 定义一个可变局部变量
    var counter = 0 
    val increment: () -> Unit = {
        // 捕获并修改局部变量
        counter++ 
    }
}
```
上面代码编译后会生成类似的代码：
```java
public final void count() {
   final Ref.IntRef counter = new Ref.IntRef();
   counter.element = 0;
   Function0 increment = (Function0)(new Function0() {
      public Object invoke() {
         this.invoke();
         return Unit.INSTANCE;
      }

      public final void invoke() {
         int var10001 = counter.element++;
      }
   });
}
```
将变量封装为 `Ref.IntRef` ： `var counter = 0 ` &rarr; `final Ref.IntRef counter = new Ref.IntRef();`

捕获全局变量：
```kotlin
class Counter {
    // 定义一个全局变量
    private var counter = 0
    
    fun count() {
        var counter = 0
        val increment: () -> Unit = {
            // 捕获并修改全局变量
            counter++
        }
    }
}
```
上面代码编译后会生成类似的代码：
```java
public final class Counter {
   private int counter;
   
   public final void count() {
      Function0 increment = (Function0)(new Function0() {
         public Object invoke() {
            this.invoke();
            return Unit.INSTANCE;
         }

         public final void invoke() {
            Counter var10000 = Counter.this;
            var10000.counter = var10000.counter + 1;
         }
      });
   }
}
```
在 increment 对象中持有外部类 Counter 对象：`Counter var10000 = Counter.this;`。

#### 示例

捕获变量延长生命周期
```kotlin
fun count() {
    var count = 0

    val runnable = Runnable { 
        println("Count: $count") 
    }

    count++ // 修改变量
    Thread(runnable).start() // 在新线程中运行 Lambda
}
```
`count` 是局部变量，但`Runnable` lambda 表达式捕获了它，让 `count` 的生命周期延长到线程结束。


### == 比较运算符

#### 定义

在 Java 中，`==` 运算符，如果比较的是基本数据类型，则表示两个变量数值是否相等；如果比较的是对象，则表示两个对象内存地址是否相同。

在 Kotlin 中，`==` 运算符用于比较两个对象内容是否相等；`===`运算符用于比较两个对象内存地址是否相同。

#### 本质

`==` 运算符用于比较两个对象时，如 `a==b`，首先会进行空检查，如果 a 对象为 `null`，然后判断 b 对象是否为 `null`，如果 b 对象也为 `null`，`a==b` 返回 `true`，否则返回 `false`；如果 a 对象不为 `null`，则会调用 a 对象的 `equals` 方法，比较内容是否相等。

`a == b` 等效于：
```java
a == null ? b == null : a.equals(b);
```

下面的代码：
```kotlin
fun main() {
    val list1 = listOf(1, 2, 3)
    val list2 = listOf(1, 2, 3)
    if(list1 == list2){

    }
    if(list1 === list2){

    }
}
```
编译后会生成类似的代码：
```java
public final void main() {
   Integer[] var2 = new Integer[]{1, 2, 3};
   List list1 = CollectionsKt.listOf(var2);
   Integer[] var3 = new Integer[]{1, 2, 3};
   List list2 = CollectionsKt.listOf(var3);
   if (Intrinsics.areEqual(list1, list2)) {
   }
   
   if (list1 == list2) {
   }

}
```
`Intrinsics.areEqual `源代码为：
```java
public static boolean areEqual(Object first, Object second) {
    return first == null ? second == null : first.equals(second);
}
```

#### 示例

字符串对象内容和地址比较：
```kotlin
fun main() {
    val str1 = "Kotlin"
    val str2 = "Kotlin"

    println(str1 == str2)  // true，内容相等
    println(str1 === str2) // true，引用相等（字符串池的优化）
}
```
普通对象内容和地址比较：
```kotlin
fun main() {
    val list1 = listOf(1, 2, 3)
    val list2 = listOf(1, 2, 3)

    println(list1 == list2)  // true，内容相等
    println(list1 === list2) // false，引用不同
}
```

#### 比较运算符本质

| 表达式 | 本质 |
| --- | --- |
| a == b |  `a?.equals(b) ?: (b === null)` |
| a != b |  `!(a?.equals(b) ?: (b === null))` |
| a > b |  `a.compareTo(b) > 0` |
| a < b |  `a.compareTo(b) < 0` |
| a >= b |  `a.compareTo(b) >= 0` |
| a <= b |  `a.compareTo(b) <= 0` |

在 Kotlin 中，编译器会把 `==` 运算符会被转换为 `equals` 方法，把`<` 或 `>` 运算符会被转换为 `compareTo` 方法。 

### by lazy() 属性懒加载
#### 定义
`by lazy()` 用于实现属性懒加载，也就是在第一次使用该属性的时候才初始化，后续使用时都是缓存值。

```kotlin
val property: T by lazy(LazyThreadSafetyMode) { initialization logic }
```
`lazy()` 参数 `LazyThreadSafetyMode` 为线程安全模式，是一个枚举值：
-   `LazyThreadSafetyMode.SYNCHRONIZED`（默认）：线程安全，适用于多线程环境。

-   `LazyThreadSafetyMode.PUBLICATION`：允许多个线程同时初始化，但只有一个结果会被使用。

-   `LazyThreadSafetyMode.NONE`：无锁，非线程安全，适用于单线程环境。

`lazy()` 参数 `lambda 表达式{}`：用于指定初始化逻辑，并返回属性值。

#### 本质
`lazy()`函数会返回一个实现了 `Lazy<T>` 接口的对象：
```kotlin
public interface Lazy<out T> {
    //初始化并缓存lambda表达式返回的值
    public val value: T

    //是否已经初始化
    public fun isInitialized(): Boolean
}
```
下面的代码：
```kotlin
class A{
   val str by lazy(LazyThreadSafetyMode.NONE) {
       "Hello Lazy"
   }
}
```
编译后会生成类似的代码：
```java
public final class A {
   @NotNull
   private final Lazy str$delegate;

   public A() {
      this.str$delegate = LazyKt.lazy(LazyThreadSafetyMode.NONE, A::str_delegate$lambda$0);
   }

   @NotNull
   public final String getStr() {
      Lazy var1 = this.str$delegate;
      return (String)var1.getValue();
   }

   private static final String str_delegate$lambda$0() {
      return "Hello Lazy";
   }
}
```
[LazyKt.lazy](https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/jvm/src/kotlin/util/LazyJVM.kt) 源代码为：
```kotlin
public actual fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T> =
    when (mode) {
        LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)
        LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer)
        LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer)
    }
```
`SynchronizedLazyImpl`、`SafePublicationLazyImpl`、`UnsafeLazyImpl`都实现了 `Lazy<T>` 接口。

以`UnsafeLazyImpl`为例，每次调用它的 `value` 时：
```kotlin
internal class UnsafeLazyImpl<out T>(initializer: () -> T) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer
    private var _value: Any? = UNINITIALIZED_VALUE

    override val value: T
        get() {
            if (_value === UNINITIALIZED_VALUE) {
                _value = initializer!!()
                initializer = null
            }
            @Suppress("UNCHECKED_CAST")
            return _value as T
        }

    override fun isInitialized(): Boolean = _value !== UNINITIALIZED_VALUE

    override fun toString(): String = if (isInitialized()) value.toString() else "Lazy value not initialized yet."

    private fun writeReplace(): Any = InitializedLazyImpl(value)
}
```
如果 `_value === UNINITIALIZED_VALUE`，先初始化`_value = initializer!!()`（`_value`的值为lambda表达式返回的结果值），然后返回 `_value`值。
#### 示例

多线程环境的属性懒加载：
```kotlin
import kotlin.concurrent.thread
import kotlin.LazyThreadSafetyMode

val lazyValue: String by lazy(LazyThreadSafetyMode.SYNCHRONIZED) {
    println("Computing lazyValue in thread ${Thread.currentThread().name}")
    "Thread-Safe Lazy Value"
}

fun main() {
    repeat(3) {
        thread {
            println(lazyValue)
        }
    }
}

输出结果：
Computing lazyValue in thread Thread-0
Thread-Safe Lazy Value
Thread-Safe Lazy Value
Thread-Safe Lazy Value
```
### 泛型参数类型`<T>`判断：is T
#### 定义
不管是在 Java 中，还是在 Kotlin 中，泛型在运行时会被擦除，擦除意味着在运行时无法判断泛型类型。

使用 `reified` 关键字，与 `inline` 函数结合，可以在运行时判断泛型类型。

```kotlin
inline fun <reified T> isType(value: Any): Boolean {
    return value is T
}
```
`inline` 关键字修饰的函数，编译器会把每一次函数调用都换成函数实际的代码实现。

#### 本质

`reified` 泛型通过内联优化，在函数调用时将泛型替换为具体类型，保留类型信息。

下面的代码：
```kotlin
inline fun <reified T> isType(value: Any): Boolean {
    return value is T
}

fun main() {
    val value = "hello"
    val isString = isType<String>(value)
    val isNumber = isType<Number>(value)
}
```
编译后会生成类似的代码：
```java
public final boolean isType(Object value) {
   Intrinsics.checkNotNullParameter(value, "value");
   int $i$f$isType = false;
   Intrinsics.reifiedOperationMarker(3, "T");
   return value instanceof Object;
}

public final void main() {
   String value = "hello";
   int $i$f$isType = false;
   boolean isString = true;
   int $i$f$isType = false;
   boolean isNumber = value instanceof Number;
}
```
value 被编译器推断为 String 类型，所以 `isString` 值直接为 true，而 Number 类型信息被保留：`boolean isNumber = value instanceof Number;`。

#### 示例
获取 Class 对象：
```kotlin
inline fun <reified T> getClassName(): String {
    return T::class.java.simpleName
}

fun main() {
    println(getClassName<String>()) // 输出：String
    println(getClassName<List<Int>>())// 输出：List
}
```
泛型类型转换：
```kotlin
inline fun <reified T> safeCast(value: Any): T? {
    return value as? T
}

fun main() {
    val result: String? = safeCast<String>("Hello")
    println(result)  // 输出: Hello

    val failed: Int? = safeCast<Int>("Hello")
    println(failed)  // 输出: null
}
```
创建实例：
```kotlin
inline fun <reified T> createInstance(): T? {
    return T::class.java.getDeclaredConstructor().newInstance()
}

class MyClass {
}

fun main() {
    val instance: MyClass? = createInstance<MyClass>()
}
```
泛型与类型擦除的对比：
```kotlin
fun <T> checkGenericType(value: T) {
    // 编译错误：无法直接使用 is 检查泛型类型
    // if (value is T) { ... }
}

inline fun <reified T> checkReifiedType(value: Any) {
    if (value is T) {
        println("Value is of type ${T::class.simpleName}")
    }
}

fun main() {
    checkReifiedType<String>("Hello")  // 输出: Value is of type String
}
```
## 总结

Kotlin 语法糖背后是 Kotlin 编译器默默努力的结果，语法糖并不改变代码的功能和底层机制。
