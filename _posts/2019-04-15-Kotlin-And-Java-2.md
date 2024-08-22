---
layout: post
title: Kotlin 与 Java 的异同（二）
categories: [Kotlin]
description: 认识 Kotlin
keywords: Kotlin
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---
本文章只为了方便查阅。

## 1.局部函数和扩展
在重构代码时，通常把长的方法分解成更小的代码块，然后重用这些代码。但是，这样可能让代码更费解，因为以一个包含许多小方法的类告终，而且它们之间并没有明确的关系。可以更进一步地将提取的函数组合成一个内部类，这样就可以保持结构，但是这种函数需要用到大量的样板代码。

Kotlin：
新定义：在函数中嵌套提取的函数。这样既可以获得所需的结构，也无需额外的语法开销。

```
class User(val id: Int, val name: String, val address: String) {
    fun saveUser(user: User) {
        fun validate(user: User, value: String, fieldName: String) {
            if (value.isEmpty()) {
                throw IllegalArgumentException("Cant't save user ${user.id} : empty $fieldName")
            }
        }
        validate(user, user.name, "Name")
        validate(user, user.address, "Address")
    }
}
```
提取逻辑到扩展函数：

```
class User(val id: Int, val name: String, val address: String) {
    fun User.validateBeforeSave() {
        fun validate(value: String, fieldName: String) {
            if (value.isEmpty()) {
                throw IllegalArgumentException("Cant't save user $id : empty $fieldName")
            }
        }
        validate(name, "Name")
        validate(address, "Address")
    }

    fun saveUser(user: User) {
        user.validateBeforeSave()
    }
}
```

Java：

```
public class User {

    private final int id;
    private final String name;
    private final String address;

    public User(int id, String name, String address) {
        this.id = id;
        this.name = name;
        this.address = address;
    }

    public void saveUser(User user) {
        validate(user, user.name, "Name");
        validate(user, user.address, "Address");
    }

    private void validate(User user, String value, String filedName) {
        if (TextUtils.isEmpty(value)) {
            throw new IllegalArgumentException("Cant't save user " + user.id + " : empty " + filedName);
        }
    }

}
```
---
## 2.定义类继承结构 
### 接口
Kotlin：

Kotlin的接口与Java 8 中的相似：它们可以包含抽象方法的定义以及非抽象方法的实现（与 Java 8 中的默认方法类似），但它们不能包含状态。

使用 **interface 关键字**来声明一个 Kotlin  的接口：

```
interface Clickable {
    fun click()
}
```
**Kotlin 在类名后面使用冒号来代替 Java 中 extends 和 implements 关键字**。和 Java 一样，一个类可以实现任意多个接口，但是只能继承一个类。

```
class Button : Clickable {
    override fun click() {
        println("I was clicked")
    }
}
```
与 Java 中的 @override 的注解类似， **override 修饰符**用来标注被重写的父类或者接口的方法和属性。与Java不同的是，在Kotlin 中使用 override 修饰符是**强制要求**的。

接口的方法可以有一个默认实现。与 Java 8 不同的是，Java 8 中需要在这样的实现上标注 **default 关键字**。

```
interface Clickable {
    fun click()

    fun showOff() = println("I'm clickable!")
}
```
如果实现这个接口，可以重新定义 showOff 方法的行为，或者可以直接忽略它。

**如果实现的多个接口中，有相同名字的方法，可以使用与 Java 相同的关键字：super。**

新增接口Focusable：

```
interface Focusable {
    fun setFocus(b: Boolean) = println("I ${if (b) "got" else "lost"} focus .")

    fun showOff() = println("I'm focusable!")
}
```

class Button 实现 Clickable 和 Focusable 接口：

```
class Button : Clickable, Focusable {
    override fun showOff() {
        super<Clickable>.showOff()
        super<Focusable>.showOff()
    }


    override fun click() {
        println("I was clicked")
    }
}
```
在方法中实现包含方法体的接口：

> Kotlin  1.0 是以 Java 6  为目标设计的，其并支持接口中的默认方法。因此它会把每个带默认方法的接口编译成一个普通接口和一个将方法体作为静态函数的类的接口体。接口中只包含声明，类中包含了静态方法存在的所有实现。因此，需要在 Java 类中实现这样一个接口，必须为所有的方法，包括在  Kotlin 中有方法体的方法定义自己的实现。

Java：

Clickable 接口：
```
public interface Clickable {
    void click();

    default void showOff() {
        System.out.println("I'm clickable!");
    }

}
```
Focusable 接口：
```
public interface Focusable {

    default void setFocus(Boolean b) {
        if (b) {
            System.out.println("I got focus .");
        } else {
            System.out.println("I lost focus .");
        }

    }

    default void showOff() {
        System.out.println("I'm focusable!");
    }

}
```
class Button 实现 Clickable 和 Focusable 接口：

```
public class Button implements Clickable, Focusable {
    @Override
    public void click() {

    }

    @Override
    public void showOff() {
        Clickable.super.showOff();
        Focusable.super.showOff();
    }
}
```
---
### open , final 和 abstract 修饰符：默认为final
#### open 
Kotlin：
 
**Java 的类和方法默认是 open 的，Kotlin 中默认都是final的。**

**如果允许创建一个类的子类，需要使用 open 修饰符来标示这个类**。此外，需要给每一个可以被重写的属性或方法添加 open 修饰符。

```
/**
 * 这个类是 open 的，其他类可以继承它
 */
open class RichButton : Clickable {

    /**
     * 这个函数重写了一个 open 函数并且它本身同样是 open 的
     */
    override fun click() {

    }

    /**
     * 这个函数是final的：不能在子类中重写它
     */
    fun disable() {

    }


    /**
     * 这个函数是 open 的：可以在子类中重写它
     */
    open fun animate() {

    }
}
```

**如果重写一个基类或者接口的成员，重写了的成员默认是 open 的。如果想改变这一行为，可以显示地将重写的成员标注为 final 。**

```
open class RichButton : Clickable {

    final override fun click() {

    }
}
```

Java：

```
/**
 * 这个类是 open 的，其他类可以继承它
 */
public class RichButton implements Clickable {
    /**
     * 这个函数重写了一个 open 函数并且它本身同样是 open 的
     */
    @Override
    public void click() {

    }

    /**
     * 这个函数是final的：不能在子类中重写它
     */
    public final void disable() {

    }


    /**
     * 这个函数是 open 的：可以在子类中重写它
     */
    public void animate() {

    }
}
```

#### abstract
Kotlin：

在Kotlin 中，同 Java 一样，可以将一个类声明为 abstract 的，这种类不能被实例化。一个抽象类通常包含一些没有实现并且必须在子类重写的抽象成员。抽象成员始终是 open 的。所以不需要显示地使用 open 的修饰符。

```
/**
 * 这个类是抽象的：不能创建它的实例
 */
abstract class Animated {
    /**
     * 这个函数是抽象的：它没有实现必须被子类重写
     */
    abstract fun animate()

    /**
     * 抽象类中的非抽象函数并不是默认 open 的，但是可以标注为 open 的
     */
    open fun stopAnimating() {

    }

    /**
     * 抽象类中的非抽象函数并不是默认 open 的
     */
    fun animateTwice() {

    }
}
```

Java：

```
/**
 * 这个类是抽象的：不能创建它的实例
 */
public abstract class Animated {
    /**
     * 这个函数是抽象的：它没有实现必须被子类重写
     */
    abstract void animate();

    /**
     * 抽象类中的非抽象函数并默认 open 的
     */
    public void stopAnimating() {

    }

    /**
     * 抽象类中的非抽象函数并不是默认 final 的，但是可以标注为 final 的
     */
    public final void animateTwice() {

    }
}
```

**类中访问修饰符的意义**
| 修饰符 |相关成员  | 评注|
|--|--| --|
| final |不能被重写  |类中成员默认使用 |
| open |可以被重写  |需要明确地使用 |
| abstract |必须被重写  |只能在抽象类中使用：抽象成员不能有实现|
| override |重写父类或接口中的成员 |如果没有使用final 表明，重写的成员默认是开放的 |

---
### 可见性修饰符：默认为 public
Kotlin：

Kotlin 中的可见修饰性修饰符与Java 中类似。同样使用public、protected 和 private 修饰符。但是默认的可见性是不一样的：**如果省略了修饰符，声明就是public 的。**

Kotlin 提供了一个新的修饰符， **internal，标示“只在模块内部可见“**。一个模块就是一组一起编译的  Kotlin 文件。这有可能是一个Intellij IDEA 模块、一个 Eclipse  项目、一个 Maven 或 Gradle 项目 或者一组使用调用 Ant 任务进行编译的文件。

Kotlin 允许在顶层声明中使用 private 可见性，包括类、函数和属性。这些声明就会只在声明它们的文件中可见。这就是另外一种隐藏子系统实现细节的非常有用的方式。

**Kotlin 的可见性修饰符**
| 修饰符|类成员| 顶层声明|
|--|--| --|
| public（默认）|所有地方可见|所有地方可见|
| internal|模块中可见| 模块中可见|
| protected|子类中可见|——|
| private|类中可见|文件中可见|

---
### 内部类和嵌套类：默认是嵌套类
Kotlin：

**Kotlin 的嵌套类不能访问外部类的实例，除非特别地做出了要求。**

Kotlin 中没有显示修饰符的嵌套类 与 Java 中的static嵌套类是一样的。要把它变成一个内部类来持有一个外部类的引用的话需要使用**inner修饰符**。

**嵌套类和内部类在 Java 与 Kotlin 中的对应关系**

| 类A 在另一个类B中声明|在Java中| 在Kotlin中|
|--|--| --|
| 嵌套类（不存储外部类的引用）|static class A|class A
| 内部类（存储外部类的引用）| class A|inner class A|

在Kotlin 中引用外部类实例的语法与Java不同。需要使用 this@Outer 从Inner类去访问 Outer类：

```
class Outer {

    inner class Inner {
        fun getOuterReference(): Outer = this@Outer
    }
}
```
Java：

```
public class Outer {
    
    class Inner{
        Outer getOuterReference(){
            return Outer.this;
        }
    }
}
```
---
### 密封类：定义受限的类继承结构
**sealed 类。为父类添加一个 sealed 修饰符，对可能创建的子类做出严格的限制。所有的直接子类必须嵌套在父类中。**

```
sealed class Expr {
    class Num(val value: Int) : Expr()
    class Sum(val left: Expr, val right: Expr) : Expr()
}

fun eval(e: Expr): Int = when (e) {
    is Expr.Num -> e.value
    is Expr.Sum -> eval(e.left) + eval(e.right)
}
```

sealed 修饰符隐含的这个类是一个 open 类，不再需要显示地添加 open 修饰符。

> 在 Kotlin 1.0 中， sealed 功能是相当严格的。例如，所有的子类必须是嵌套的，并且子类不能创建为 data 类。 Kotlin 1.1 解除了这些限制并允许在同一个文件的任何位置定义sealed类的子类。

---

## 3. 声明一个带默认构造方法或属性的类
Kotlin 区分**主构造方法**（通常是主要而简洁的初始化类的方法，并且**在类体外部声明**）和**从构造方法**（**在类体内部声明**）。同样也允许在**初始化语句块**中添加额外的初始化逻辑 。

### 初始化类：主构造方法和初始化语句块
Kotlin：

```
class User constructor(val _nickname: String) { //带一个参数的构造方法
    val nickname: String

    init { // 初始化语句块
        nickname = _nickname
    }
}
```
constructor 和 init 是两个新的关键字。**constructor 关键字**用来开始一个主构造方法或从构造方法的声明。**init 关键字** 用来引入一个初始化语句块。这种语句块包含了在类被创建时执行的代码，并会与主构造方法一起使用。因为主构造方法有语法规则，不能包含初始化代码，所以要使用初始化语句块。也可以在一个类中声明多个初始化语句块。

上面的代码也可以简化为：

```
class User(val nickname: String) {
    
}
```
构造方法参数也可以有默认值：

```
class User(val nickname: String, val isSubscribed: Boolean = true) {

}
```

> 如果所有的构造方法参数都有默认值，编译器会生成一个额外的不带参数的构造方法来使用所有的默认值。这可以让 Kotlin 使用库时变得更简单，因为可以通过无参构造方法来实例类。

**如果一个类具有父类，主构造方法同样需要初始化父类。**

```
class TwitterUser(nickname: String) : User(nickname) {

}
```
**如果没有给一个类声明任何的构造方法，将会生成一个不做任何事情的默认构造方法：**

```
open class Button
```

```
class RadioButton : Button()
```
**注意与接口的区别**：接口没有构造方法，所以在实现一个接口的时候，不需要在父类型列表中它的名称后面再加上括号。

**如果想要确保类不会被其他的代码实例化，必须把构造方法标记为 private。**

```
class Secretive private constructor(){
}
```

**private  构造方法的替代方案**

> 在Java 中，可以通过使用 private 构造方法禁止实例化这个类来标示一个更通用的意思：这个类是一个静态实用工具成员的容器或是单例的。Kotlin 针对这种目的具有内建的语言级别的功能。可以实用顶层函数作为静态实用工具。想要表示单例，可以使用对象声明。

Java：

```
public class User {

    private String nickname;
    private boolean isSubscribed = true;

    public User(String nickname) {
        this.nickname = nickname;
    }
}
```

```
public class Twitter extends User {
    public Twitter(String nickname) {
        super(nickname);
    }
}
```

```
public class Secretive {
    
    private Secretive(){}
    
}
```
---
### 构造方法：用不同的方式来初始化父类
Kotlin：
如果父类中有多个构造方法，在 一个构造方法中使用this关键字来调用类中的另一个构造方法。

```
class MyButton : View {
    constructor(context: Context?) : this(context, null)
    constructor(context: Context?, attrs: AttributeSet?) : super(context, attrs)
}
```
Java：

```
public class MyButton extends View {
    public MyButton(Context context) {
        this(context, null);
    }

    public MyButton(Context context, AttributeSet attrs) {
        super(context, attrs);
    }
}
```

---
### 实现在接口中声明的属性
Kotlin：
在 Kotlin 中，**接口可以包含抽象属性声明**。

```
interface User {
    val nickname: String
}
```
实现 User 接口的类需要提供一个取得 nickname 值的方式。接口并没有说明这个应该存在到一个支持字段还是通过 getter 来获取。接口本身并不包含任何状态，因此只有实现这个接口的类在需要的情况下会存储这个值。

实例：以不同的方式实现接口中的抽象属性。

使用override来实现抽象属性：
```
class PrivateUser(override val nickname: String) : User {
}

```
通过自定义 getter 实现抽象属性：
```
class SubscribingUser(val email: String) : User {
    override val nickname: String
        get() = email.substringBefore('@')
}
```
将抽象属性与其他函数建立关联：
```
class FacebookUser(val accountId:Int):User{
    override val nickname: String
        get() = getFackbookName(accountId)
}
```
---

### 通过 getter 或 setter 访问支持字段
Kotlin：
在setter 的函数体中，使用了特殊的标识符 field 来访问支持字段的值。在 getter 中，只能读取值；而在setter 中，既能读取它也能修改它。

```
class User(val name: String) {
    var address: String = "unspecified"
        set(value: String) {
            println("""Address was changed for $name:"$field" -> "$value".""".trimIndent())
            field = value
        }
}
val user = User("Alice")
user.address = "Elsenheimerstrasse 47, 80687 Muenchen"
```
注意，可以只重定义可变属性的一个访问器。

Java：

```
public class User {

    private String nickname;
    private boolean isSubscribed = true;
    private String address;

    public User(String nickname) {
        this.nickname = nickname;
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }
}
```
---
### 修改访问器的可见性
Kotlin：

访问器的可见性默认与属性的可见性相同。但是如果需要可以通过在 get 和 set 关键字前放置可见性修饰符的方式来修饰它。

```
class LenthCounter {
    var counter: Int = 0
        private set

    fun addWord(word: String) {
        counter += word.length
    }
}
```
Java :

```
import android.support.annotation.NonNull;

public class LenthCounter {

    private int counter = 0;

    public void addWord(@NonNull String word) {
        counter += word.length();
    }

    public int getCounter() {
        return counter;
    }
}
```
---
## 4.编译器生成的方法：数据类和类委托
### 数据类：自动生成通用方法的实现
如果为类添加 **data 修饰符**，会自动生成 toString，equals 和 hashCode 方法。

equals 和 hashCode 方法会将**所有主构造方法中**声明的属性纳入考虑。生成的 equals 法会检测所有的属性的值是否相等。 hashCode 方法会返回一个根据所有属性生成的哈希值。**没有在主构造方法中**声明的属性将不会加入到相等性检查和哈希值计算中去。

```
data class Client(val name: String, val postalCode: Int) {

}
val client1 = Client(name = "wangjiang", postalCode = 100001)
val client2 = Client(name = "wangjiang", postalCode = 100001)
client1.equals(client2)：结果为true
```

#### 数据类和不可变性：copy()方法
数据类的属性推荐使用val，但也可以使用var，使用val主要是让数据类的实例不可变。这样做的好处是，如果将这样的实例作为 HashMap 或者类似容器的键，在被用作键对象加入到容器后就不可被修改，容器将一直处于一种有效状态。另外，在多线程中，不可变也不用担心其他线程修改对象的值。

为了让使用不可变对象的数据变得更容易，Kotlin 编译器为它们多生成了一个方法：**一个允许 copy 类的实例的方法，并在copy的同时修改某些属性的值。创建副本通常是修改实例的好选择：副本有着单独的声明周期而且不会影响代码中引用原始实例的值。**

```
val client3 = client1.copy(postalCode = 100003)
```
---
### 类委托：使用 "by" 关键字
Kotlin：

Kotlin 默认将类视作为 final 的。只有那些设计成可扩展的类可以被继承。当使用这样的类时，需注意修改与派生类兼容。

 **by 关键字**将接口的实现委托到另一个对象。
 

```
class CountingSet<T>(val innerSet: MutableCollection<T> = HashSet<T>()) : MutableCollection<T> by innerSet { //将MutableCollection 的实现委托给 innerSet
    var objectAdded = 0
    override fun add(element: T): Boolean {
        objectAdded++
        return innerSet.add(element)
    }

    override fun addAll(elements: Collection<T>): Boolean {
        objectAdded += elements.size
        return innerSet.addAll(elements)
    }
}
val cset = CountingSet<Int>()
cset.addAll(listOf(1, 1, 2))
println("${cset.objectAdded} objects were added, ${cset.size} remain")
3 objects were added , 2 remain
```
Java :

```
public class CountingSet<T> extends HashSet<T> {

    private int objectsAdded = 0;

    @Override
    public boolean add(T t) {
        objectsAdded++;
        return super.add(t);
    }

    @Override
    public boolean addAll(Collection<? extends T> c) {
        objectsAdded += c.size();
        return super.addAll(c);
    }

    public int getObjectsAdded() {
        return objectsAdded;
    }
}
```
---

## 5."Object" 关键字：将声明一个类与创建一个实例结合起来
Kotlin：

**object 关键字**定义一个类并同时创建一个实例。使用的场景有：

 - **对象声明**是定义单例的一种方式。
 - **伴生对象**可以持有工厂方法和其他与这个类相关，但在调用时并不依赖类实例的方法。它们的成员可以通过类名来访问。
 - **对象表达式**用来代替Java的匿名内部类。

### 对象声明：创建单例
 Kotlin：
对象声明将类声明与该类的单一实例声明结合到了一起。

```
object Payroll {

    val allEmployees = arrayListOf<Person>()

    fun calculateSalary() {
        for (person in allEmployees) {
            
        }
    }
}
Payroll.allEmployees.add(Person())
Payroll.calculateSalary()
```
与类一样，一个对象声明也可以包含属性、方法、初始化语句块等的声明。唯一不允许的就是构造方法（包括主构造方法和从构造方法）。与普通类的实例不同，对象声明在定义的时候就立即构建了，不需要在代码的其他地方调用构造方法。因此，为对象声明定义一个构造方法是没有意义的。

对象声明同样可以继承自类和接口：

```
object CaseInsensitiveFileComparator : Comparator<File> {
    override fun compare(o1: File, o2: File): Int {
        return o1.path.compareTo(o2.path, ignoreCase = false)
    }
}
CaseInsensitiveFileComparator.compare(File("/User"),File("/user"))
```
也**可以在任何可以使用普通对象的地方使用单例对象**：

```
        val files = listOf(File("/Z"), File("/a"))
        files.sortedWith(CaseInsensitiveFileComparator)
```
也**可以在类中声明对象**：

```
data class Person(val name: String, private val age: Int) {

    object NameComparator : Comparator<Person> {
        override fun compare(o1: Person, o2: Person): Int = o1.name.compareTo(o2.name)
    }
}
```
在Java中使用Kotlin 对象：

> Kotlin 中的对象声明被编译成了通过静态字段来持有它的单一实例的类，这个字段名字始终都是 INSTANCE。因此，要从  Java 代码使用 Kotlin对象，可以通过访问静态的INSTANCE字段：`CaseInsensitiveFileComparator.INSTANCE.compare(file1,file2)`

Java 单例：

```
public final class Singleton {

    private Singleton() {

    }

    private static class SingletonHolder {
        private static Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }

}
```

---
### 伴生对象
Kotlin：

伴生对象是一个声明在类中的普通对象。**它可以有名字，实现一个接口或者有扩展函数或属性。**

实例：

```
data class Person(val name: String, private val age: Int) {

    companion object Loader {
        fun fromJSON(jsonText: String): Person {

        }
    }
   
}
val person = Person.Loader.fromJSON(your json string)
```
在大多数情况下，通过包含伴生对象的类的名字来引用伴生对象，所以不必关心它的名字。但是也可以指明名字（像上面例子一样）。如果省略了伴生对象的名字，**默认的名字将会分配为 Companion。**

#### 在伴生对像中实现接口
和其它对象一样，伴生对象也可以实现接口。

```
    
interface JSONFactory<T> {
    fun fromJSON(jsonText: java.lang.String): T
}

data class Person(val name: String, private val age: Int) {

    companion object Loader : JSONFactory<Person> { //实现接口的伴生对象
        override fun fromJSON(jsonText: java.lang.String): Person {

        }

    }
    
}
```

#### 伴生对象扩展
如果类有一个伴生对象，可以通过在其上定义扩展函数来做到和Java 静态方法相同的效果。

```
data class Person(val name: String, private val age: Int) {

   companion object {
       
   }
    
}

fun Person.Companion.fromJSON(jsonText:String):Person{
    
}

Person.fromJSON()
```

---

### 对象表达式：改变写法的匿名内部类
object 关键字不仅仅能用来声明单例式的对象，还能用来声明匿名对象。匿名对象替代了 Java 中的匿名内部类的用法。

实例：

```
class MyApplication : Application() {

    override fun onCreate() {
        super.onCreate()
        
        var activityAliveCount = 0
        registerActivityLifecycleCallbacks(object : ActivityLifecycleCallbacks {
            override fun onActivityResumed(activity: Activity?) {
            }

            override fun onActivityStarted(activity: Activity?) {
            }

            override fun onActivityDestroyed(activity: Activity?) {
                activityAliveCount--
            }

            override fun onActivitySaveInstanceState(activity: Activity?, outState: Bundle?) {
            }

            override fun onActivityStopped(activity: Activity?) {
            }

            override fun onActivityCreated(activity: Activity?, savedInstanceState: Bundle?) {
                activityAliveCount++
            }

            override fun onActivityPaused(activity: Activity?) {
            }

        })
    }
}
```
对象表达式声明了一个类并创建了该类的一个实例，但是并没有给这个类或是实例分配一个名字。通常来说，它们都是不需要名字的，因为会将这个对象用作一个函数调用的参数。

如果需要给对象分配一个名字，可以将其存储到一个变量中：

```
class MyApplication : Application() {

    override fun onCreate() {
        super.onCreate()
        
        registerActivityLifecycleCallbacks(activityLifecycleCallbacks)
    }

    var activityAliveCount = 0
    val activityLifecycleCallbacks = object : ActivityLifecycleCallbacks {
        override fun onActivityResumed(activity: Activity?) {
        }

        override fun onActivityStarted(activity: Activity?) {
        }

        override fun onActivityDestroyed(activity: Activity?) {
            activityAliveCount--
        }

        override fun onActivitySaveInstanceState(activity: Activity?, outState: Bundle?) {
        }

        override fun onActivityStopped(activity: Activity?) {
        }

        override fun onActivityCreated(activity: Activity?, savedInstanceState: Bundle?) {
            activityAliveCount++
        }

        override fun onActivityPaused(activity: Activity?) {
        }

    }
}
```

**与Java 匿名内部类只能扩展一个类或实现一个接口不同，Kotlin 的匿名对象可以实现多个接口或者不实现接口。**

**注意：与对象声明不同，匿名对象不是单例的。每次对象表达式被执行都会创建一个新的对象实例。**

**与 Java 的匿名类一样，在对象表达式中的代码可以访问创建它的函数中的变量。但是与Java 不同，访问并没有被限制在 final 变量，还可以在对象表达式中修改变量的值。**

 
**注意：**
> 对象表达式在需要在匿名对吸纳更中重写多个方法时是最有用的。如果只需要实现一个单方法的接口（就像 Runnable ，可以将实现写作函数字面值（lambda）并依靠 Kotlin 的 SAM 转换（把函数字面值转换成单抽象函数接口的实现）。
