---

layout: post
title: Kotlin 与 Java 的异同（一）
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

## Kotlin简介

Kotlin是一种针对Java 平台的新编程语言。Kotlin简洁、安全、务实，并且专注于与Java代码的互操作性。它几乎可以用在现在Java使用的任何地方：服务端开发、Android应用等等。Kotlin 可以很好地和所有现存的Java库和框架一起工作，而且性能和Java旗鼓相当。

Kotlin 特点：

- Kotlin 是**静态类型语言并支持类型推导**，允许维护正确性与性能的同时保持源代码的简洁。
- Kotlin **支持面向对象和函数式两种编程风格**，通过头等函数使更高级别的的抽象成为可能，**通过支持不可变值简化了测试和多线程开发**。
- 在服务端应用程序中它可以工作得很好，全面支持所有现存的 Java 框架，为常见的任务提供新工具，如生成 HTML和持久化。
- 在 Android上它也可以工作，这得益于紧凑的运行时、对Android API 特殊的编译器支持以及丰富的库，为常见Android开发任务提供了Kotlin 友好的函数。
- 它是免费和开源的，全面支持主流的IDE 和构建系统。
- Kotlin 是务实的、安全的、简洁的，与Java可互操作，意味着它专注于使用已经证明过的解决方案处理常见任务，**防止常见的像NullPointerException这样的错误**，支持紧凑和易读的代码，以及提供与Java无限制的集成。

补充说明：

1.静态类型语言：**所有表达式的类型在编译期已经确定了**，而编译器就能验证对象是否包含了你想访问的方法或者字段。

2.函数式编程：

- 头等函数：**把函数（一小段行为）当作值使用，可以用变量保存它，把它当作参数传递，或者当作其他函数的返回值**。
- 不可变性：使用不可变对象，这保证了它们的状态在其创建之后不能再变化。
- 无副作用：使用的纯函数。此类函数在输入相同时会产生同样的结果，并且不会修改其他对象的状态，也不会和外面的世界交互。

## Kotlin与Java的异同

### 1.函数

kotlin：

```
    fun main(args: Array<String>) {
        println("Hello, world!")
    }
```

- 关键字 fun 用来声明一个函数。

- 参数的类型写在它的名称后面。

- 函数可以定义在文件的最外层，不需要把它放在类中。

- **数组就是类**。

- 使用 println 代替了 System.out.println。

- 和许多其他现代语言一样，可以省略每行代码结尾的分号。
  
  表达式函数体：

```
    fun max(a: Int, b: Int): Int = if (a > b) a else b
```

语句和表达式：

 在 Kotlin 中，if 是表达式，而不是语句。语句和表达式的区别在于，表达式有值，并且能作为另一个表达式的一部分使用；而语句总是包围着它的的代码块中的顶层元素，并且没有自己的值。在Java 中，所有的控制结构都是语句。**而在Kotlin中，除了循环(for 、while、和 do/while）以外大多数控制结构都是表达式**。这种结构控制结构和其他表达式的能力让你可以简明扼要地表示许多常见的模式。

**另一方面，Java中的赋值操作是表达式，在Kotlin中反而变成了语句**。这有助于避免比较和赋值之间的混淆，而这种混淆是常见的错误来源。

 Java：

```
    public void main(String[] args) {
        System.out.println("Hello, world!");
    }
```

### 2. 变量

#### 变量类型

Kotlin：

```
    val a = 5 //可以不显示声明变量类型
    val a: Int = 5 //也可以显示声明变量类型
```

Java：

```
int a = 5;  //必须显示声明变量类型
```

#### 可变变量

Kotlin：

```
    var answer = 0
    answer = 1
```

**var** ： 可变引用。这种变量的值可以被改变。这种声明对应的是普通(非 final）的 Java 变量。

Java：

```
    int answer = 0
    answer = 1
```

非 final 修饰即可。

#### 不可变变量

Kotlin：

```
    val answer = 0
```

**val**： 不可变引用。使用 val 声明 的变量不能在初始化之后再次赋值。它对应的是 Java 的 final 变量。

默认情况下，应该尽可能地使用 val 关键字 来声明所有的 Kotlin 变量，仅在必要的时候换成var。使用不可变引用、不可变对象及无副作用的函数让代码更接近函数式编程风格。

在定义了 val  变量的代码块执行期间， val 变量只能进行唯一一次初始化。但是，**如果编译器能确保只有唯一一条初始化语句被执行，可以根据条件使用不同的值来初始化它**：

```
        val message: String
        if (canPerformOperation()) {
            message = "Success"
        } else {
            message = "Failed"
        }
```

**尽管 val 引用自身是不可变的，但是它指向的对象可能是可变的。**

```
        val languages = arrayListOf("Java")
        languages.add("Kotlin")
```

Java：

```
  final int answer = 0
```

 final 修饰即可。

### 3. 类和属性

#### 有参数的构造方法

Kotlin：

```
class Person(val name: String)
```

Java：

```
public class Person {
    private final String name;

    public Person(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}
```

#### setter 和 getter

Kotlin：

```
class Person {

    val name: String //只读属性：生成一个字段和一个简单的getter

    var isMarried: Boolean // 可写属性：一个字段、一个getter 和 一个setter

}
```

访问属性：

```
        val person = Person()
        person.name //只读，不可以修改值
        person.isMarried //可读写
```

自定义getter：

```
class Person(val age: Int) {

    val name: String? = null

    var isMarried: Boolean = false


    val isOld: Boolean
        get() {
            return age > 60
        }
}
```

访问isOld：

```
        val person = Person(50)
        person.name = "小明"
        person.isMarried = true
        person.isOld
```

Java：

```
public class Person {

    private int age;

    private final String name;

    private boolean isMarried;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public boolean isMarried() {
        return isMarried;
    }

    public void setMarried(boolean married) {
        isMarried = married;
    }

    public boolean isOld() {
        return this.age > 50;
    }
}
```

### 4. 枚举和"when"

#### 枚举

Kotlin：

```
enum class Color(val r: Int, val g: Int, val b: Int) {
    RED(255, 0, 0),
    ORANGE(255, 165, 0),
    YELLOW(255, 255, 0),
    GREEN(0, 255, 0),
    BLUE(0, 0, 255),
    INDIGO(75, 0, 130),
    VIOLET(238, 130, 238)
}
```

Kotlin 用了一个**enum class** 两个关键字，而 Java 只有 enum 一个关键字。

Java：

```
public enum Color {
    RED(255, 0, 0),
    ORANGE(255, 165, 0),
    YELLOW(255, 255, 0),
    GREEN(0, 255, 0),
    BLUE(0, 0, 255),
    INDIGO(75, 0, 130),
    VIOLET(238, 130, 238);

    private int r;
    private int g;
    private int b;

    Color(int r, int g, int b) {
        this.r = r;
        this.g = g;
        this.b = b;
    }

    public int getR() {
        return r;
    }

    public int getG() {
        return g;
    }

    public int getB() {
        return b;
    }
}
```

#### when

Kotlin：

```
    fun getWarmth(color: Color): String {
        when (color) {
            Color.RED, Color.ORANGE, Color.YELLOW -> return "warm"
            Color.GREEN -> return "neutral"
            Color.BLUE, Color.INDIGO, Color.VIOLET -> return "cold"
            else -> return ""
        }
    }
```

Kotlin 中的when结构对应Java 中的switch语句。

Java：

```
    public String getWarmth(Color color) {
        switch (color) {
            case RED:
            case ORANGE:
            case YELLOW:
                return "warm";
            case GREEN:
                return "neutral";
            case BLUE:
            case INDIGO:
            case VIOLET:
                return "cold";
            default:
                return "";
        }
    }
```

### 5."while" 和 "for" 循环

#### "while" 循环

while 和 do-while 循环与 Java 循环是一样的：

```
while (condition) {
/*...*/
}

do { 

}  while (condition){

}
```

#### "for" 循环

Kotlin：

使用`..`运算符 来表示区间：

```
        val oneToTen = 1..10

        for (i in 1..100){
            println(i)
        }
```

```
fun isNotDigit(c: Char) = c !in '0'..'9'
```

**区间是包含或者闭合的，意味着第二个值始终是区间的一部分。**

迭代 map：

```
        val binaryResp = TreeMap<Char, String>()
        for (c in 'A'..'Z') {
            val binary = Integer.toBinaryString(c.toInt())
            binaryResp[c] = binary
        }
        for ((letter, binary) in binaryResp) {
            println("$letter = $binary")
        }
```

使用下标迭代：

```
        val list = arrayListOf("10", "11", "111")
        for ((index, element) in list.withIndex()) {
            println("$index : $element")
        }
```

Java：

```
        for (int i = 1; i <= 100; i++) {
            System.out.println(i);
        }

        TreeMap<Character, String> binaryResp = new TreeMap<>();
        Character[] characters = {'A', 'B', 'C','D','Z'};
        for (Character c : characters) {
            String binary = Integer.toBinaryString(c);
            binaryResp.put(c, binary);
        }
        Set<Map.Entry<Character, String>> entries = binaryResp.entrySet();
        for (Map.Entry<Character, String> entry : entries) {
            System.out.println(entry.getKey() + "=" + entry.getValue());
        }

        List<String> list = Arrays.asList("10", "11", "111");
        for (int i = 0, size = list.size(); i < size; i++) {
            System.out.println(i + ":" + list.get(i));
        }
```

### 6.异常

#### 抛出异常

Kotlin：

```
        val percentage = 50
        if (percentage !in 0..100)
            throw IllegalArgumentException("A percentage value must be between 0 and 100: $percentage")
```

不必使用 new 关键字来创建异常实例。

与 Java 不同的是，**Kotlin 中的 throw 结构是一个表达式，能作为另一个表达式的一部分使用**：

```
    val percentage =
        if (number in 0..100) {
            number
        } else {
            throw IllegalArgumentException("A percentage value must be between 0 and 100: $percentage")
        }
```

Java：

```
        int percentage = 50;
        if (percentage < 0 || percentage > 100) {
            throw new IllegalArgumentException("A percentage value must be between 0 and 100: " + percentage);
        }
```

#### "try" "catch" 和 "finally"

Kotlin：

```
    fun readNumber(reader: BufferedReader): Int? {
        try {
            val line = reader.readLine()
            return Integer.parseInt(line)
        } catch (e: NumberFormatException) {
            return null
        } finally {
            reader.close() //finally 的作用和 Java 中的一样
        }
    }
```

**Kotlin 不区分受检异常和未受检异常。** 在这里没有处理readLine方法抛出的 IOException 异常。

"try" 也可以作为表达式：

```
    fun readNumber(reader: BufferedReader): Int? {
        return try {
            val line = reader.readLine()
            Integer.parseInt(line)
        } catch (e: NumberFormatException) {
            null
        } finally {
            reader.close()
        }
    }
```

Java：

```
    public Integer readNumber(BufferedReader reader) throws IOException {
        try {
            String line = reader.readLine();
            return Integer.parseInt(line);
        } catch (NumberFormatException e) {
            return null;
        } finally {
            reader.close();
        }
    }
```

IOException 是一个受检异常。**在 Java  中必须显示地处理。必须声明函数能抛出的所有受检异常。如果调用另外一个函数，需要处理这个函数的受检异常，或者声明函数也能抛出这些异常。**

### 7.创建集合

Kotlin：

```
        val set = hashSetOf(1, 7, 53) //创建 HashSet 集合

        val arrayList = arrayListOf(1, 7, 53) //创建 List 集合

        val map = hashMapOf(1 to "one", 2 to "two", 3 to "three") //创建 HashMap 集合

        set.javaClass // Kotlin 的 javaClass 等价于 Java  的 getClass()

        arrayList.last() // 获取列表的最后一个

        arrayList.max() //得到数字列表的最大值
```

Java：

```
        Set<Integer> set = new HashSet<Integer>(); //创建 HashSet 集合
        set.add(1);
        set.add(7);
        set.add(53);

        List<Integer> arrayList = new ArrayList<>(); //创建 List 集合
        arrayList.add(1);
        arrayList.add(7);
        arrayList.add(53);

        Map<Integer, String> map = new HashMap<>(); //创建 HashMap 集合
        map.put(1, "one");
        map.put(2, "two");
        map.put(3, "three");

        set.getClass();  //Java  的 getClass() 等价于 Kotlin 的 javaClass 

        if (arrayList.size() > 0) // 获取列表的最后一个
            arrayList.get(arrayList.size() - 1);

        Collections.max(arrayList, new Comparator<Integer>() { //得到数字列表的最大值
            @Override
            public int compare(Integer o1, Integer o2) {
                if (o1 > o2) {
                    return 1;
                } else if (o1 == o2) {
                    return 0;
                }
                return -1;
            }
        });
```

### 8.函数

#### 命名参数

Kotlin：

假设现在有一个函数，它的作用是在集合元素中添加分割符号，然后将集合转化为字符串。

```
    fun <T> joinToString(collection: Collection<T>, separator: String, prefix: String, postfix: String): String {

        val result = StringBuffer(prefix)

        for ((index, element) in collection.withIndex()) {
            if (index > 0) result.append(separator)
            result.append(element)
        }

        result.append(postfix)

        return result.toString()

    }
```

在调用函数时，显示地标明一些参数的名称，避免参数混淆：

```
        val arrayList = arrayListOf(1, 7, 53)
        joinToString(arrayList, separator = ";", prefix = "[", postfix = "]")
```

Java：

```
    public <T> String joinToString(Collection<T> collection, String separator, String prefix, String postfix) {
        StringBuilder sb = new StringBuilder(prefix);

        Iterator<T> iterator = collection.iterator();
        int i = 0;
        while (iterator.hasNext()) {
            if (i > 0) sb.append(separator);
            sb.append(iterator.next());
            i++;
        }

        sb.append(postfix);
        return sb.toString();
    }
```

在调用函数时，标明参数的名称：

```
        List<Integer> arrayList = new ArrayList<>();
        arrayList.add(1);
        arrayList.add(7);
        arrayList.add(53);
        joinToString(arrayList, /* separator */";",/* prefix */ "[", /* postfix */ "]");
```

#### 默认参数

Kotlin：

声明函数的时候，指定参数的默认值，**避免创建重载函数**：

```
    fun <T> joinToString(
        collection: Collection<T>,
        separator: String = ";",
        prefix: String = "",
        postfix: String = ""
    ): String
```

在调用函数时，可以用所有参数来调用，也可以省略掉部分参数：

```
        joinToString(arrayList, ";", "", "")
        joinToString(arrayList)
        joinToString(arrayList, ";")
```

**当使用常规的调用语法时，必须按照函数声明中定义的参数顺序来给定参数，可以省略的只有在末尾的参数。如果使用命名参数，可以省略中间的一些参数，也可以以任意顺序只给定需要的参数**：

```
        joinToString(arrayList, prefix = "[",postfix = "]")
```

Java：

```
    public <T> String joinToString(Collection<T> collection) {
        return joinToString(collection, ";", "", "");
    }

    public <T> String joinToString(Collection<T> collection, String separator) {
        return joinToString(collection, separator, "", "");
    }

    public <T> String joinToString(Collection<T> collection, String separator, String prefix, String postfix) {
         //...
    }
```

因为Java 没有默认值的概念，从Java 中调用 Kotlin 函数的时候，必须显示地指定所有参数值。如果需要从 Java 代码中做频繁的调用，而且希望它能对 Java 的调用者简便，可以用 **@JvmOverloads** 注解它。

```
    @JvmOverloads
    fun <T> joinToString(
        collection: Collection<T>,
        separator: String = ";",
        prefix: String = "",
        postfix: String = ""
    ): String
```

此时，编译器就会生成如下重载函数：

```
    public <T> String joinToString(Collection<T> collection) {

    }

    public <T> String joinToString(Collection<T> collection, String separator) {

    }

    public <T> String joinToString(Collection<T> collection, String separator, String prefix) {

    }

    public <T> String joinToString(Collection<T> collection, String separator, String prefix, String postfix) {

    }
```

**每个重载函数的默认参数值都会被省略。**

#### 静态工具类

##### 顶层函数（静态函数）

Kotlin：

Kotlin 中的新定义：**顶层函数**，也就是把函数直接放到代码文件的顶层，不用从属于任何的类。这些文件顶层的函数依然是包内的成员，如果需要从包外访问它，则需要 import ，但不再需要额外包一层。

实例：把 joinToString 直接放到 strings 的包中，创建一个名为 Join.kt 的文件。

```
package com.example.kotlin.strings

fun <T> joinToString(
    collection: Collection<T>,
    separator: String = ";",
    prefix: String = "",
    postfix: String = ""
): String {

    val result = StringBuffer(prefix)

    for ((index, element) in collection.withIndex()) {
        if (index > 0) result.append(separator)
        result.append(element)
    }

    result.append(postfix)

    return result.toString()

}
```

在 Kotlin 中调用顶层函数：

```
        com.example.kotlin.strings.joinToString(arrayList, ";", "", "")
```

在 Java 中调用顶层函数：

```
        com.example.kotlin.strings.JoinKt.joinToString(arrayList, /* separator */";",/* prefix */ "[", /* postfix */ "]");
```

要改变包含Kotlin 顶层函数的生成的类的名称，需要为这个文件添加 @JvmName 的注解，将其放到这个文件的开头，位于包名的前面：

```
@file:JvmName("StringFunctions")

package com.example.kotlin.strings

fun <T> joinToString(
    collection: Collection<T>,
    separator: String = ";",
    prefix: String = "",
    postfix: String = ""
): String {
   //......
}
```

然后在 Java 中再调用顶层函数，类型需要改变：

```
        com.example.kotlin.strings.StringFunctions.joinToString(arrayList, /* separator */";",/* prefix */ "[", /* postfix */ "]");
```

顶层函数对应 Java 中的静态函数。

Java：

```
public class JoinJava {

    public static <T> String joinToString(Collection<T> collection, String separator, String prefix, String postfix) {
        StringBuilder sb = new StringBuilder(prefix);

        Iterator<T> iterator = collection.iterator();
        int i = 0;
        while (iterator.hasNext()) {
            if (i > 0) sb.append(separator);
            sb.append(iterator.next());
            i++;
        }

        sb.append(postfix);
        return sb.toString();
    }
}
```

##### 顶层属性（静态变量）

Kotlin:

Kotlin 中的新定义：**顶层属性**，和顶层函数一样，属性也是放到文件的顶层。

```
@file:JvmName("StringFunctions")

package com.example.kotlin.strings

const val LANGUAGE_KOTLIN: String = "Kotlin"
```

在 Kotlin 中调用顶层函数：

```
        com.example.kotlin.strings.LANGUAGE_KOTLIN
```

在 Java 中调用顶层函数：

```
        com.example.kotlin.strings.StringFunctions.LANGUAGE_KOTLIN;
```

顶层属性对应 Java 中的静态属性。

Java：

```
    public static final String LANGUAGE_KOTLIN  = "Kotlin";
```

### 9.可变参数

Kotlin：

参数通过 **vararg** 来修饰：

```
    fun convertNumbersToList(vararg values: Int) {
        println(values)
    }

    convertNumbersToList(1, 2, 3, 5)
```

Java：

```
    public void convertNumbersToList(int ...values){
        System.out.println(values);
    }

    convertNumbersToList(1,2,3,4);
```

### 10.字符串和正则表达式

Kotlin：

**在三重引号中的字符串，不会对任何字符进行转义，包括反斜杠。**

实例：使用正则表达式解析文件路径。

文件路径格式：

```
"/users/wangjiang/koltin-book/chapter.adoc"
"/users/wangjiang/koltin-book/" 目录
"chapter" 文件名
".adoc" 扩展名
```

函数解析文件路径：

```
    fun parsePath(path: String) {
        val regex = """(.+)/(.+)\.(.+)""".toRegex()
        val matchResult = regex.matchEntire(path)
        if (matchResult != null) {
            val (directory, filename, extension) = matchResult.destructured
            println("Dir:$directory,name:$filename,ext:$extension")
        }
    }
```

**三重引号字符串除了避免转义字符以外，还可以包含任何字符，包括换行符。**

```
        val kotlinLogo = """|  //
            .| //
            .|/ \
        """.trimMargin(".")
        Log.d("parsePath",kotlinLogo)

        |  //
        | //
        |/ \
```

Java：

函数解析文件路径：

```
    public void parsePath(String path) {
        Pattern pattern = Pattern.compile("(.+)/(.+)\\.(.+)");
        Matcher matcher = pattern.matcher(path);
        if (matcher.find()) {
            Log.d("parsePath", "Dir:" + matcher.group(1) + ",name:" + matcher.group(2) + ",ext:" + matcher.group(3) );
        }
    }
```
