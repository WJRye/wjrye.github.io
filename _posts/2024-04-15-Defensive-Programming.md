---
layout: post
title: 防御式编程
categories: [Architect]
description: 防御性编程的目的始终是提高代码质量，降低线上出现问题的概率。
keywords: 防御式编程
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
topmost: false
---

## 前言

#### 编写高质量代码有什么用？
作为一名程序猿，或软件开发工程师，或技术工人，首要任务就是编写好代码(编写高质量代码)。那么编写好代码有什么用呢？靠编写好代码可以维持你的工作（基础），可以去影响你的同事和领导（让你同事和领导从代码中认为你是一个技术能力靠谱的人）（进阶），那么继续下去，你可能参与团队或部门或公司重要的项目或事情（高级），再下去，你就可以在团队和部门中提高你的技术影响力，从而你更有机会获得更多的资源（时间和人力）和更多的成就（升职和加薪）。当然，也有可能在你工作的环境中，编写好代码和编写坏代码结果一样。那么，尽早离开，去追寻你想要的编写好代码的工作环境。

编写高质量代码重要的一环：防御式编程。

#### 防御式编程是什么？

防御式编程（Defensive Programming）是一种编程实践或编程范式，通过在代码中添加防御性的逻辑和错误处理机制，来降低软件出现错误或异常的概率，从而提高软件的正确行、稳定性、安全性，让软件更健壮。

其实，防御式编程就是在编写代码的时候，假设代码周围的环境（任何输入和输出等）都是不可信的，然后采取预防性措施避免潜在的错误或异常。


#### 防御式编程的目的是什么？

目的很简单，就是保护自己（没有crash，没有离谱的错误，没有甩锅等🐶），不管你写的代码是给自己用还是给别人用，在任何情况下都需要保证代码的正确性和可靠性-健壮。

## 防御式编程

防御性编程该怎么做？在《代码大全2》中提到了：

1. 保护程序免遭非法输入数据的破坏
2. 断言
3. 错误处理技术
4. 异常
5. 隔离程序，使之包容由错误造成的损害
6. 辅助调试的代码
7. 确定在产品代码中该保留多少防御式代码
8. 对防御式编程采取防御的姿态

根据这八个方面，以及个人在实际项目中积累的一些真实经验。可以通过以下几点来做防御性编程：
1. 输入验证
2. 输出验证
3. 异常
4. 日志
5. 测试保障
6. ab 开关

通过这几点，能有效的预防线上出现 crash。

### 输入验证

当在编写一个方法的时候，外部输入参数的值，可以是任意的、随意的、不确定的值。那么，编写方法首先需要校验参数值。

检验参数值的准则大概有：
* 是否为 null
* 是否在有效范围内
* 是否符合预期的格式

下面简单介绍几种方式来检验方法参数值。

#### 基础类型验证
1. 普通验证

```java
public void checkIntArgs(int a){
  //正负数验证
  if(a<0) return;
  
  //进度验证
  if(a>=0 && a<=100){
  
  }
}

```
2.  使用自定义注解`@IntDef`、`@StringDef`限定常量参数
```java

@IntDef({WeekDays.MONDAY, WeekDays.TUESDAY, WeekDays.WEDNESDAY, WeekDays.THURSDAY, WeekDays.FRIDAY, WeekDays.SATURDAY, WeekDays.SUNDAY})
@Retention(value = RetentionPolicy.SOURCE)
public @interface WeekDays {
    int MONDAY = 1;
    int TUESDAY = 2;
    int WEDNESDAY = 3;
    int THURSDAY = 4;
    int FRIDAY = 5;
    int SATURDAY = 6;
    int SUNDAY = 7;
}


public class Day {
    private @WeekDays int currentDay;

    public Day() {
    }

    @WeekDays
    public int getCurrentDay() {
        return currentDay;
    }
    //如果 currentDay 不是 WeekDays，那么编译器就会报错
    public void setCurrentDay(@WeekDays int currentDay) {
        this.currentDay = currentDay;
    }
}
```
3. 使用库 `androidx.annotation`的 `@IntRange`、`@FloatRange`、`@Size`限定参数值范围
```java
public void setDayOfMonth(@IntRange(from = 1, to = 31) int day) {
   //说明 day 值必须在 1到31 范围内，否则编译器就会报错
}


public void setProgress(@IntRange(from = 0, to = 100) int progress) {
  //说明 progress 必须在 0到100 范围内，，否则编译器就会报错
}


public void getLocationInWindow(@Size(2) int[] outLocation){
  //说明 outLocation 的大小为2
}
```
4. 使用库 `androidx.annotation`的`@IntegerRes`、`@AnimRes`、`@StringRes`等注解限定资源类型

```java
public void setDay(@IntegerRes int resId){
   //说明 resId 是资源值，否则编译器就会报错
}
```

5. float 和 double 可能会精度问题，比较和计算不能直接使用
```java
public int compareFloat(float a, float b) {
    return Float.compare(a, b);
}

public int compareDouble(double a, double b) {
    return Double.compare(a, b);
}

public float computeFloatAdd(float a, float b) {
    return BigDecimal.valueOf(a).add(BigDecimal.valueOf(b)).floatValue();
}

public double computeDoubleAdd(double a, double b) {
    return BigDecimal.valueOf(a).add(BigDecimal.valueOf(b)).doubleValue();
}
```
6. 包装类型有欺骗性，不能拿来就用
```java
public void checkIntegerArgs(Integer args) {
    if (args == null) {
        return;
    }
    int value = args;
}

public void checkBooleanArgs(Boolean args) {
    if (args == null) {
        return;
    }
    boolean value = args;
}
```
```java
public void testCheckIntegerArgs() {
    List<Integer> args = new ArrayList<>();
    args.add(-1);
    args.add(0);
    args.add(1);
    checkIntegerArgs(args);
}

public void checkIntegerArgs(@NonNull List<Integer> args) {
    int removeValue = 1;
    args.remove(removeValue);
    //移除的并不是1，而是0。因为调用的是args.remove(int index)，不是args.remove(Object o)
}
```
小结：多用注解约束参数，Android 提供了注解库 `androidx.annotation`。阅读 Activity 或 Fragment 源码，可以发现有很多注解库 `androidx.annotation`相关使用。

#### 对象类型验证
1. 使用库 `androidx.annotation`的`@Nullbale`、`@NonNull`注解限定对象可空和不可空
```java

public void sendMessage(@Nullable Person person) {
    //可空对象
}

public void sendMessage(@NonNull Person person) {
   //不可空对象
}
```
如果是使用 kotlin 的 `?` 来限定对象可空和不可空，最好也添加注解。因为混合开发项目中， 存在Java 和 Kotlin 代码互相调用。

或者使用 `java.util.Optional`对传入参数对象包装：
```java
public void sendMessage(Person person) {
    String name = Optional.ofNullable(person).map(new Function<Person, String>() {

        @Override
        public String apply(Person person) {
            return person.getName();
        }
    }).orElse("unknown");
}
```

3. 对象可读可写
```java
public class Person {
    private String name;

    public Person(String name) {
        this.name = name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return this.name;
    }
}

public void setPersonaName(@NonNull Person person) {
    //拿到 Person 对象，首先看 Person 源码，判断其可读、可写
    //外部和内部都可更改 person.name，需注意 perosn.name 的准确性
    person.setName("xxx");
}
```
对于 `sendMessage`方法中的 Person 参数对象，使用时需要注意：
   * 拿到 Person 对象，首先看 Person 源码，判断其可读、可写
   * 外部和内部都可更改 person.name，那么 perosn.name 可能缺乏准确性

2. 对象可读不可写
```java
public final class Person {
    private final String name;

    public Person(String name) {
        this.name = name;
    }
    
    public String getName() {
        return this.name;
    }
}
    public void setPersonaName(Person person) {
        // 不提供 setName 方法，如果没有和定义 Person 的作者协商，又要修改 name，不能直接去 Person 中添加 setName 方法，因为这可能导致外部使用的 person.name 失去准确性
        person.getName();
    }
```
或者提供 Builder
```java
public final class Person {
    private String name;

    private Person() {
    }

    public String getName() {
        return this.name;
    }

    public class Builder {
        private String name;

        public Builder setName(String name) {
            this.name = name;
            return this;
        }

        public Person build() {
            Person person = new Person();
            person.name = this.name;
            return person;
        }
    }
}

public void setPersonaName(Person person) {
    // 不提供 setName 方法
    person.getName();
}

```
对于 `sendMessage`方法中的 Person 参数对象，使用时需要注意：Person 对象不提供 setName 方法，**如果没有和定义 Person 的作者协商，又要修改 name，是不能直接去 Person 中添加 setName 方法**，因为这可能导致外部使用的 person.name 失去准确性。

3. 对象拷贝，防止受到外部修改的影响
```java
public final class Period {
    private final Date start;
    private final Date end;

    public Period(Date start, Date end) {
        this.start = start;
        this.end = end;
    }

    public Date getStart() {
        return this.start;
    }

    public Date getEnd() {
        return this.end;
    }
}

Data 可以修改它的 time: start.setTime(System.currentTimeMillis());所以上面代码需要更改为：

public Period(Date start, Date end) {
    this.start = new Date(start.getTime());
    this.end = new Date(end.getTime());
}
```
表面看 Date start 和 Date end 对象都是可读不可写，但其实 start 和 end 对象是可写的：`start.setTime(System.currentTimeMillis())`，所以要对对象进行保护性拷贝。

5. 使用对象副本
```java
public void cloneList(@NonNull List<String> list) {
    List<String> newList = new ArrayList<String>(list);
}

public void unmodifiableList(@NonNull List<String> list) {
    List<String> newList = Collections.unmodifiableList(list);
}

public void cloneArray(@NonNull int[] arr) {
    int[] newArr = Arrays.copyOf(arr, arr.length);
}
```
如果不明确外部传入的集合或数组对象是否可修改，那么就使用该集合或数组对象的副本。Kotlin 中区分了集合的可读和可写：`Collection`，`MutableCollection`。

4. 消化不了的文件
```java
@Nullable
public String getStringFromFile(@NonNull String filePath) {
    //如果 filePath 对应的文件过大，就不要读取
    File file = new File(filePath);
    long max = 10 * 1024 * 1024L;
    if (!file.exists() || file.length() == 0L || file.length() > max) {
        //消化不了
        return null;
    }
}

@Nullable
public Bitmap getBitmapFromFile(@NonNull String filePath) {
    //如果 filePath 对应的图片过大，就不要读取
    BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeFile(filePath, options);
    if (options.outWidth == 0 || options.outHeight == 0) {
        return null;
    }
    int max = 1080 * 1920;
    int size = options.outWidth * options.outHeight;
    if (size > max) {
        return nulll;
    }
}
```
从文件中读取字符串内容，或从图片文件中获取Bitmap，或读取音频、视频文件，或压缩和解压文件，或上传文件等场景，都需要注意文件是否过大。

5. 消化不了的大对象
```java
public void checkCollectionArgs(@NonNull Collection<String> collection){
    int max = 128;
    if (collection.size() > max) {
        return;
    }
}

public void checkArrayArgs(@Size(128) @NonNull int[] arr) {
    int max = 128;
    if (arr.length > max) {
        return;
    }
}

public void checkStringArgs(@Size(min = 1, max = 256) @NonNull String str) {
    int max = 256;
    if (str.length() > max) {
        return;
    }
}
```
传入大对象，远超过了该方法承受的能力，所以需要确定方法所能接受的对象有效范围。

### 输出验证
方法的输出值，主要是保证其正确性。

返回值的准则大概有：

* 避免返回空值：如果方法可能返回空值，应该在文档注释中明确说明，并考虑通过返回可选类型（Optional）、空对象模式（Null Object Pattern）等方式来处理。
* 一致性：返回值应该在不同的调用情况下保持一致性。即使方法执行的环境或参数发生变化，返回值的语义和类型也应该保持不变。
* 准确性：给方法一个输入参数 x，那么它的输出值就是 y = f(x)，方法的逻辑在任何输入条件下都能产生正确的结果。


1. 使用库 `androidx.annotation`的 `@NonNull` 和`@Nullable`注解限定返回对象可空和不可空
```java
@NonNull
public Person getPerson(int id){

}

@Nullable
public Person getPerson(int id){

}
```
或 使用 `java.util.Optional` 返回可选类型
```java
public Optional<Person> getPerson(int id) {
    Person person = db.getPersonById(id);
    return Optional.ofNullable(person);
}
```
或 空对象

```java
@NonNull
public Person getPerson(int id) {
    Person person = db.getPersonById(id);
    return person != null ? person ? new Person();
}
```
另外，关于空对象模式（Null Object Pattern），主要用于多态：

```java
// 定义形状接口
interface Shape {
    void draw();
}

// 具体的实现类 - 圆形
class Circle implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a circle.");
    }
}

// 具体的实现类 - 矩形
class Rectangle implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a rectangle.");
    }
}

// 空对象类
class NullShape implements Shape {
    @Override
    public void draw() {
        // 空对象的默认行为是什么，这里可以根据实际需求定义
        System.out.println("No shape to draw.");
    }
}

```

```java
// 形状工厂类
class ShapeFactory {
    // 获取形状对象
    @NonNull
    public static Shape getShape(String shapeType) {
        if (shapeType == null) {
            return new NullShape(); // 返回空对象
        } else if (shapeType.equalsIgnoreCase("Circle")) {
            return new Circle(); // 返回圆形对象
        } else if (shapeType.equalsIgnoreCase("Rectangle")) {
            return new Rectangle(); // 返回矩形对象
        } else {
            return new NullShape(); // 返回空对象
        }
    }
}
```

2. 返回有效默认值，而不是null
```java
public List<Cats> getCats() {
    List<Cats> cats = zoo.getCats();
    return cats != null ? cats : Collections.emptyList();
}

private final static Cats[] EMPTY_CAT_ARRAY = new Cats[0];

public Cats[] getCats() {
    Cats[] cats = zoo.getCats();
    return cats != null ? cats : EMPTY_CAT_ARRAY;
}

public String getCatName(int catId) {
    String name = db.getNameById(catId);
    return name != null ? name : "";
}
```
3. 同样，可以使用库`androidx.annotation`的注解 `@IntDef`、`@StringDef` 或 `@IntRange`、`@FloatRange`、`@Size`限定返回值范围

4. 使用库 `androidx.annotation` 的`@RequiresPermission`注解限制权限，返回值受权限授予的影响（比如方法中有访问文件时，注意权限）
```java
@RequiresPermission(Manifest.permission.KILL_BACKGROUND_PROCESSES)
public static Set<String> getAllBackgroundProcesses() {
    ActivityManager am = (ActivityManager) Utils.getApp().getSystemService(Context.ACTIVITY_SERVICE);
    List<ActivityManager.RunningAppProcessInfo> info = am.getRunningAppProcesses();
    Set<String> set = new HashSet<>();
    if (info != null) {
        for (ActivityManager.RunningAppProcessInfo aInfo : info) {
            Collections.addAll(set, aInfo.pkgList);
        }
    }
    return set;
}
```
6. 使用库 `androidx.annotation` 的 `@WorkerThread` 、 `@UIThread` 、 `@MainThread` 或 `@AnyThread` 注解限制线程
 ```java
@WorkerThread
@NonNull
public static List<Resource> getUserAllResources(File jsonFile) {
    String str = FileUtil.readSdCardJsonFile(jsonFile.getAbsolutePath());
    return getUserAllResources(str);
}
```
或者使用 kotlin `suspend` 限制线程
````kotlin
suspend fun uploadFile(path: String):String{
   return withContext(Dispatchers.IO) {
   }
}
````

### 异常

首先，熟悉一下 Java 语言异常。

Java 语言，异常分为Exception 和 Error，Exception 和 Error 类都继承自 `Throwable` 类。

* **Exception（程序可恢复）**：表示程序可以处理的异常，可以捕获并且可恢复。遇到这类异常，应该尽可能处理异常，使程序恢复运行，而不应该随意终止异常。

* **Error（程序不可恢复）**：一般指虚拟机相关的问题，如系统崩溃，虚拟机错误，内存空间不足，方法调用栈溢出等。这于这类错误，Java编译器不去检查也会导致应用程序中断，仅靠程序本身无法恢复和预防，遇到这样的错误，建议程序终止。

Exception 又分为运行时异常和受检查的异常：

* **运行时异常**：如空指针，参数错误等。
* **受检查异常**：这类异常如果没有try/catch语句也没有throws抛出，编译不通过。

编写一个方法的时候，主要是处理运行时异常和受检异常。

对于处理异常的准则大概有：
* 精确捕获异常：只捕获能够处理的异常类型，避免捕获过于宽泛的异常，这样才能更精准地定位和处理问题。
* 适当处理异常：根据具体情况适当地处理异常，可以是通过修复问题、提供默认值、向上层抛出异常或记录日志等方式。
* 不要忽略异常：不要在 catch 块中不加思考地空实现或打印空日志，这样会隐藏问题并使其更难以调试和解决。
* 异常处理与业务逻辑分离：将异常处理与业务逻辑分离，避免在业务逻辑中直接处理异常，这样可以使代码更清晰、可读性更高。
* 统一异常处理：在合适的地方进行统一的异常处理，例如在顶层统一处理未捕获的异常，并将异常信息记录下来，以便后续分析和处理。
* 良好的异常信息：在捕获异常时提供清晰、具体的异常信息，包括异常类型、发生位置、异常原因等，便于后续排查问题。
* 异常的抛出和传递：在适当的时候抛出异常并将异常传递给调用者，让调用者决定如何处理异常，避免在方法内部过度处理异常。


1. 调用他人提供的方法或不信任的方法，最好try-catch

他人提供的方法，如果没有仔细阅读其源码，那么就得小心使用，否则可能触发隐藏的炸弹，如下面通用工具类 `FileUtils.java` 提供的方法 `long sizeOf(File file)`：
```java
public static long sizeOf(@NonNull File file) {
    if (!file.exists()) {
        String message = file + " does not exist";
        throw new IllegalArgumentException(message);
    }
    if (file.isDirectory()) {
        return sizeOfDirectory(file);
    } else {
        return file.length();
    }
}
```
然后在自己的方法中直接使用：

```java
public void uploadVideo(@NonNull String filePath){
    long fileSize = FileUtils.sizeOf(new File(filePath));
}
```
这在线上运行，只能祈求不要触发这个炸弹，所以**使用他人提供的方法前，最好阅读一下源码或查看他人是怎么使用的，然后看是否有必要try-catch 一下**：
```java
public void uploadVideo(@NonNull String filePath){
    try {
        long fileSize = FileUtils.sizeOf(new File(filePath));
    } catch (IllegalArgumentException e) {
        Log.e(TAG, e.getMessage(), e);
    }
}
```

2. 输出并上报捕获到的异常信息

确保异常信息能被 app 日志库收集到，或在条件允许下，主动将异常信息上报到公司的 apm 平台。

**要主动获取异常信息，不要被动获取异常信息，被动时可能为时已晚**。
```java
public void uploadVideo(@NonNull String filePath){
    try {
        long fileSize = FileUtils.sizeOf(new File(filePath));
    } catch (IllegalArgumentException e) {
        //app 日志库收集日志
        MyLog.e(TAG, e.getMessage(), e);
        //主动上报异常信息到 apm 平台
        CrashReporter.postCaughtException(new Exception("Upload Video Failed.",e));
    }
}
```
3. 处理异常，声明异常，统一异常

处理异常时，处理具体异常，并详细描述相关异常信息，有多个异常时，可以统一处理异常，如参看系统方法
`FragmentFactory#instantiate` 的异常处理：
```java
/**
 * Create a new instance of a Fragment with the given class name. This uses
 * {@link #loadFragmentClass(ClassLoader, String)} and the empty
 * constructor of the resulting Class by default.
 *
 * @param classLoader The default classloader to use for instantiation
 * @param className The class name of the fragment to instantiate.
 * @return Returns a new fragment instance.
 * @throws Fragment.InstantiationException If there is a failure in instantiating
 * the given fragment class.  This is a runtime exception; it is not
 * normally expected to happen.
 */
@NonNull
public Fragment instantiate(@NonNull ClassLoader classLoader, @NonNull String className) {
    try {
        Class<? extends Fragment> cls = loadFragmentClass(classLoader, className);
        return cls.getConstructor().newInstance();
    } catch (java.lang.InstantiationException e) {
        throw new Fragment.InstantiationException("Unable to instantiate fragment " + className
                + ": make sure class name exists, is public, and has an"
                + " empty constructor that is public", e);
    } catch (IllegalAccessException e) {
        throw new Fragment.InstantiationException("Unable to instantiate fragment " + className
                + ": make sure class name exists, is public, and has an"
                + " empty constructor that is public", e);
    } catch (NoSuchMethodException e) {
        throw new Fragment.InstantiationException("Unable to instantiate fragment " + className
                + ": could not find Fragment constructor", e);
    } catch (InvocationTargetException e) {
        throw new Fragment.InstantiationException("Unable to instantiate fragment " + className
                + ": calling Fragment constructor caused an exception", e);
    }
}
```
4. 处理不了或可不处理的异常向外抛
```java
/**
 * 从文件中读取字符串内容
 * @param path 文件地址
 *
 * @return 读取失败时返回空字符串
 */
public static String fileToString(String path) throws IOException {
    return FileUtils.readFileToString(new File(path), "UTF-8");
}
/**
 * 将字符串写入文件
 *
 * @param filename 文件名称
 * @param string 字符串内容
 */
public static void stringToFile(String filename, String string) throws IOException {
    FileWriter out = new FileWriter(filename);
    try {
        out.write(string);
    } finally {
        out.close();
    }
}
```
5. 在开发阶段尽量暴露异常，让程序崩溃，排查出异常具体原因，然后修改代码，让代码更健壮
```java
public static String fileToString(String path){
    try {
        return FileUtils.readFileToString(new File(path), "UTF-8");
    } catch (IOException e) {
        //遇到异常，在开发阶段让程序崩溃，以消除问题case
        throw new RuntimeException(e);
    }
}
```
### 日志

在编写的代码中，可以分阶段：开发阶段、测试阶段、线上阶段添加一些关键日志。添加日志的目的是：了解程序运行时的状态和行为，从而诊断和调试问题，然后做出修正和改进以保证其正确性。

添加日志的主要作用是：故障排查、性能分析、行为跟踪。

* 故障排查：在程序出现错误或异常时快速定位问题
* 性能分析：通过记录程序的运行时间、资源消耗等信息进行性能分析，或发现程序中的性能瓶颈
* 行为跟踪：了解程序的行为和执行流程，方便问题排查和复现


在添加日志时，一般需要记录的信息有：

* 错误和异常信息：记录程序出现的错误和异常信息，包括异常类型、堆栈信息等
* 关键操作和状态变化：记录关键操作的执行情况和状态变化，例如数据库操作、网络请求、用户交互等
* 性能指标：记录程序的性能指标，例如运行时间、内存占用、CPU利用率等
* 调试信息：记录调试信息，包括变量值、方法调用栈、条件分支执行情况等

注意：在添加日志时，根据具体情况选择合适的日志级别和格式，避免过多或过少地记录日志信息，以确保日志的有效性和实用性。同时，注意保护用户隐私和敏感信息，避免将敏感信息记录在日志中。

#### 开发日志

开发阶段的日志主要是故障排查，所以尽量详细一点，并让日志级别升高。但在上线之前，尽量清除开发日志，最多保留 error 级别日志。

比如新写一个方法：保存 bitmap 到文件，开发关心 bitmap 是否回收，存储文件夹是否存在，存储遇到异常等
```java
public static boolean saveBitmap(@NonNull Bitmap bitmap, @NonNull String dir, @NonNull String name, @IntRange(from = 0, to = 100) int quality) {
    if (bitmap.isRecycled()) {
        MyLog.e(TAG, "bitmap is recycled");
        return false;
    }
    if (TextUtils.isEmpty(dir)) {
        MyLog.e(TAG, "dir is empty");
        return false;
    }
    if (TextUtils.isEmpty(name)) {
        MyLog.e(TAG, "name is empty");
        return false;
    }
    File dirFile = new File(dir);
    if (!dirFile.exists()) {
        boolean ret = dirFile.mkdirs();
        if (!ret) {
            MyLog.e(TAG, "create dir failed. dir=" + dir);
            return false;
        }
    }
    File file = new File(dirFile, name);
    OutputStream outputStream = null;
    try {
        boolean ret = file.createNewFile();
        if (!ret) {
            return false;
        }
        outputStream = new FileOutputStream(file);
        return bitmap.compress(Bitmap.CompressFormat.PNG, quality, outputStream);
    } catch (FileNotFoundException e) {
        MyLog.e(TAG, "FileNotFound:" + file.getAbsolutePath(), e);
    } catch (IOException e) {
        MyLog.e(TAG, e.getMessage(), e);
    } finally {
        if (outputStream != null) {
            try {
                outputStream.close();
            } catch (IOException e) {
                MyLog.e(TAG, "close stream failed");
            }
        }
    }
    return false;
}
```


#### 测试日志
测试阶段的日志主要是测试人员关注的日志信息，除了故障排查，性能分析和行为追踪相关日志外，可能还有一些业务上下文，或项目环境相关日志。但在上线之前，尽量清除测试日志。

比如文件下载，测试人员关注下载所有状态，下载开始，下载进度，下载速度，下载暂停，下载重试，下载成功，下载失败，下载取消。
```kotlin
private val downloadListener = object : DownloadListener {
    override fun onStart(taskId: String) {
        super.onStart(taskId)
        MyLog.e(TAG,"onStart:taskId=$taskId")
    }

    override fun onLoading(taskId: String, speed: Long, totalSize: Long, loadedSize: Long, progress: Int) {
        super.onLoading(taskId, speed, totalSize, loadedSize, progress)
        MyLog.e(TAG,"onLoading:taskId=$taskId, speed=$speed, totalSize=$totalSize, loadedSize=$loadedSize, progress=$progress")
    }

    override fun onPause(taskId: String, totalSize: Long, loadedSize: Long) {
        super.onPause(taskId, totalSize, loadedSize)
        MyLog.e(TAG,"onPause:taskId=$taskId, totalSize=$totalSize, loadedSize=$loadedSize")
    }

    override fun onRetry(taskId: String, retryTimes: Int) {
        super.onRetry(taskId, retryTimes)
        MyLog.e(TAG,"onRetry:taskId=$taskId, retryTimes=$retryTimes")
    }

    override fun onFinish(taskId: String, dir: String?, name: String?) {
        super.onFinish(taskId, dir, name)
        MyLog.e(TAG,"onFinish:taskId=$taskId, dir=$dir, name=$name")
    }

    override fun onError(taskId: String, errorCodes: List<Int>?, totalSize: Long, loadedSize: Long) {
        super.onError(taskId, errorCodes, totalSize, loadedSize)
        MyLog.e(TAG,"onError:taskId=$taskId, errorCodes=$errorCodes, totalSize=$totalSize, loadedSize=$loadedSize")
    }

    override fun onCancel(taskId: String) {
        super.onCancel(taskId)
        MyLog.e(TAG,"onCancel:taskId=$taskId")
    }
}
```
#### 线上日志
线上阶段的日志主要是排查问题，要简短关键。

比如业务代码，从文件中读取json字符串并转换为对象，如果获取到的对象为空，要快速排查问题所在，是字符串编码问题，还是 json 数据问题等。
```java
@WorkerThread
@Nullable
private Cat getCat(@NonNull String jsonPath){
    if (TextUtils.isEmpty(jsonPath)) {
        return null;
    }
    File jsonFile = new File(jsonPath);
    try {
        String json = readFileToString(jsonFile, "UTF-8");
        return JSON.parseObject(json, Cat.class);
    } catch (IOException e) {
        MyLog.e(TAG, "read file failed: jsonPath=" + jsonPath + ", fileSize=" + jsonFile.length(), e);
        return null;
    } catch (JSONException e) {
        MyLog.e(TAG, "parse json failed: " + json, e);
        return null;
    }
}

/**
 * 读取文件内容转换为字符串
 *
 * @param file     文件
 * @param encoding 编码方式
 * @return 文件内容字符串
 * @throws IOException                  I/O异常
 * @throws UnsupportedEncodingException 字符串编码异常
 */
@NonNull
public static String readFileToString(@NonNull File file, @Nullable String encoding) throws IOException {
    Charset charset = null;
    try {
        charset = encoding == null ? Charset.defaultCharset() : Charset.forName(encoding);
    } catch (Exception e) {
        throw new UnsupportedEncodingException("Unsupported encoding " + encoding);
    }
    byte[] buff = readFileToByteArray(file);
    return new String(buff, charset);
}
```

### 测试保障
当完成一个 feature，一般流程是开发人员使用测试用例进行自我测试，然后测试人员进行冒烟测试、全量测试、集成测试、验收测试等验证，最后上线。在这个过程中，feature 质量或者说代码质量的保障，不应该完全交给测试人员来保障。作为开发人员，只有自己对代码最熟悉，所以完成 feature 后，可以自我进行一些质量保障检查。

#### 单元测试

单元测试理论上的作用是验证代码的正确性和可靠性，发现潜在的 bug 和错误，以提高代码的质量。但目前项目开发实际情况是：要么没有单元测试，要么单元测试写了就扔了，这可能与我们的项目环境，产品迭代周期，技术氛围有关系。不过，对于个人来说，写单元测试总是好的，特别是在做技术需求，比如开发上传组件、下载组件等基础通用组件，或者做核心代码更改和重构时，能帮助自己更了解需求，更了解方法输入输出，更了解模块功能使用场景，进而不断的构思和修改逻辑，让逻辑在代码中和自己脑袋中都越来越清晰。

AndroidStudio对 Android 项目的提供了 [本地单元测试和插桩测试](https://developer.android.com/studio/test/test-in-android-studio?hl=zh-cn)。

这里不介绍如何去做本地单元测试和插桩测试。

单元测试需要测试策略：良好的测试策略应围绕代码的不同路径和边界。在最基本的层面上，可以将测试分类为三种场景：**成功路径、错误路径和边界情况**。
> -   成功路径：成功路径测试（也称为理想路径测试）侧重于测试正向流的功能。正向流是指不存在任何异常或错误情况的流。与错误路径和边界情况场景相比，创建成功路径场景的详尽列表是很容易的，因为它们关注的是应用的预期行为。
> 
> -   错误路径：错误路径测试侧重于测试负向流的功能，即检查应用如何响应错误情况或无效用户输入。确定所有可能的错误流是一项极具挑战性的任务，因为如果未实现预期行为，则会有许多可能的结果。（一项一般性建议是列出所有可能的错误路径，针对这些错误编写测试，并随着您发现不同的场景而不断改进单元测试。）
> 
> -   边界情况：边界情况侧重于测试应用中的边界条件。
> 

创建单元测试时需要注意**测试准则**：
> 有效的单元测试通常具有以下四个属性：
> 
> -   有针对性：测试应侧重于某个单元，例如一段代码，通常是某个类或方法。测试应有针对性并侧重于验证单段代码的正确性，而不是同时验证多段代码。
> -   易于理解：当您阅读代码时，代码应当简单且易于理解。开发者应当能够立即一目了然地了解测试背后的意图。
> -   确定性：应保持一致的通过或失败结果。如果您运行测试多次且没有更改任何代码，测试应得到相同的结果。测试应避免不可靠性，也就是在未修改代码的情况下，不得在一个实例中测试失败，而在另一个实例中测试通过。
> -   独立性：测试不需要任何人为互动或设置，即可独立运行。

**单元测试需要关心代码覆盖率**：Android Studio 为本地单元测试提供了测试覆盖率工具，用于跟踪单元测试所覆盖的应用代码的百分比和区域。

代码覆盖率有分析报告，如：
-   单元测试覆盖的方法百分比：到目前为止所编写的测试覆盖了 8 种方法中的 7 种。这占方法总数的 87%。
-   单元测试覆盖的行数百分比：编写的测试覆盖了 41 行代码中的 39 行。这占代码行的 95%。

注意：**代码覆盖率只是用来查找测试未覆盖的代码部分，而不是利用代码覆盖率来衡量代码质量。**

参考文献
- [本地单元测试和插桩测试](https://developer.android.com/studio/test/test-in-android-studio?hl=zh-cn)
- [为 ViewModel 编写单元测试](https://developer.android.com/codelabs/basic-android-kotlin-compose-test-viewmodel?hl=zh-cn#0)
- [Android单元测试研究与实践](https://tech.meituan.com/2015/12/24/android-unit-test.html)

#### monkey测试

Android Monkey 测试是一种用于自动化应用程序的压力测试和随机测试的工具。Monkey 测试工具会在应用程序上执行各种随机事件，如点击、滑动、按键等，以模拟用户的操作行为。这些随机事件的目的是发现应用程序中的潜在问题，如崩溃、ANR（应用程序未响应）和内存泄漏等。

在使用 Monkey 测试时，可以指定一些参数来控制测试的持续时间、事件数量等。Monkey 测试生成的事件序列是随机的，因此它可以发现应用程序中可能存在的各种问题，尤其是在用户操作方式无法预测的情况下。

Monkey 测试通过使用 adb 命令来执行：

```shell
$ adb shell monkey [options] <event-count>
```

比如：延迟500毫秒执行一次，共执行1000次
```shell
$ adb shell monkey -p pkgname --throttle 500 -v -v -v 1000
```

Monkey 测试官网介绍：[UI/​Application Exerciser Monkey](https://developer.android.com/studio/test/other-testing-tools/monkey)

#### 开启严格模式

Android StrictMode 是一个开发工具，能帮助开发人员检测应用程序中可能存在的一些性能和违规问题，如网络请求、磁盘操作、主线程耗时等。

可以发现的问题分为以下几类：

1.  磁盘读写：检测应用中是否存在在主线程进行磁盘读写操作的情况。
2.  网络请求：检测应用中是否存在在主线程进行网络请求的情况。
3.  主线程耗时操作：检测应用中是否存在在主线程进行耗时操作的情况，比如数据库查询、大量计算等。
4.  内存泄漏：检测应用中是否存在内存泄漏的情况，比如Activity或Fragment未正确释放等。

如果项目开启了严格模式，在 Logcat 中查看是否有自己编写的代码相关的日志，如：

```java
androidx.fragment.app.strictmode.WrongFragmentContainerViolation

androidx.fragment.app.strictmode.SetUserVisibleHintViolation

android.os.strictmode.LeakedClosableViolation:

//......其它
```
如果是比较严重的问题，应该修复。

StrictMode 官网介绍：[StrictMode](https://developer.android.com/reference/android/os/StrictMode)

#### 不保活测试

Android 中的"不保活"测试指的是在开发者选项中设置不保留活动选项，勾选这个选项后，应用程序退到后台会被系统销毁，进入前台后会重新启动，所以这主要测试应用程序是否能在重新启动的情况下正确地保存和恢复状态。

在进行不保活测试时，可以测试应用程序的启动速度、数据的保存和恢复、用户体验等方面的问题。

#### 静态代码检查

项目开发中，静态代码检查是有必要的，它能防止出现低端错误问题。但静态代码检查可能由于项目环境的原因，无法顺利进行，尤其需要要做的还是增量静态代码检查。

首先，Android Studio 中标黄色警号的代码一定要逐一查看并修改。

其次，在 Android Studio 中，选中类或包，进行 Analyze &rarr; Inspect Code 分析检查。

最后，使用 [detect](https://detekt.dev/) 或 [pmd](https://docs.pmd-code.org/latest/index.html) 静态代码工具辅助进行代码检查。

之前写了一篇文章：[Python 封装 detekt 和 pmd 命令- 增量静态代码检查](https://juejin.cn/post/7336831338119020559)，感兴趣的可以查看。

#### code review

code review 是代码上线前的最后卡口，为了防止代码出现线上问题，把控好代码质量，每个团队都期望有严格的 code review。但在实际开发中，要么没有 code review，要么 code review 只是一个过场。如果 code reivew 能进入版本迭代的标准流程节点，那么 code review 是很容易能够在团队中培养起来的。

这里不介绍 code review 的好处和坏处，以及如何进行 code review。如果感兴趣，可以查看文档[what-is-code-review](https://about.gitlab.com/topics/version-control/what-is-code-review/)。

在实际项目开发中，通常采取结对 code review 是最好的实践。首先，选择一个同事做你的 backup，当然你也做他的 backup。但 backup 前提是：
1. 你们熟悉彼此的业务
2. 你们的技术水平相当
3. 你们的编程习惯，代码风格相似

在每次合并代码前，你俩坐在一起，一个负责解说代码，这部分代码是做什么用的，为什么这样做，把代码的逻辑和思路告诉你的 reviewer，reviewer 负责对代码提出建议和反馈，直到你们双方对这次更改代码达成共识，然后才合并代码。

### ab开关
现在每个公司都有 apm 平台，当观测到线上代码出现了问题，可以通过 ab 开关及时止损，这是非常重要的一环。

只有上面的操作步骤都失效了，ab 开关才能出现，所以添加 ab 开关得慎重。但预留有 ab 开关总是好的，那么什么场景下适合添加 ab 开关呢？

适合添加 ab 开关的场景（不能完全掌控的场景），如：
1. 修改影响多个业务方，不能把控
2. 需求较大（迭代周期横跨多个版本），第一次上线
3. 重构代码
4. 在不熟悉的老业务中增加新的 feature
5. 产品或数据分析或老板比较关心的业务更改：登录、推送、数据埋点等

添加 ab 开关需要做到代码隔离，如果命中满足条件（app 版本、系统版本、机型等）的用户，就走新的逻辑，否则走老的逻辑。添加 ab 开关要能够控制代码分流，无论是修改还是新增代码。

```java
public void runAB() {
    boolean isNew = config();
    if (isNew) {
        //走新的逻辑
        newCode();
    } else {
        //走老的逻辑
        oldCode();
    }
}
```

## 总结
上面从输入验证、输出验证、异常、日志、测试保障、ab 开关方面，分析了防御性编程应该怎么做，防御性编程的目的始终是提高代码质量，降低线上出现问题的概率。只有代码没有问题了，才有更多的时间和精力去做更重要的事情。所以，专注于写高质量代码，将形成一个正反馈，带来良性循环。

