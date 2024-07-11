---
layout: post
title: Android 中自定义注解和注解解析
categories: [Android]
description: 注解主要用在编译期间对代码进行扫描或在运行期间通过反射执行相应的操作。
keywords: Android, Annotation
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 前言

注解在很多第三方库中都有被用到，比如常用的一些库：

1. [EventBus](https://github.com/greenrobot/EventBus)：事件发布-订阅总线，使组件之间的通信解耦。
2. [butterknife](https://github.com/JakeWharton/butterknife) ：View注入框架，使View的绑定自动化。
3. [Retrofit](http://square.github.io/retrofit/)：网络加载库。
4. [Junit](https://junit.org/junit5/docs/current/user-guide/) 和 [AndroidJUnitRunner](https://developer.android.google.cn/training/testing/junit-runner)：单元测试框架。

常见的注解：`@see @param @return` ，`@Nullable @NonNull`，` @Override`，`@LayoutRes @DimenRes`等

注解是 Java 1.5 版本引入的，它不同于注释，是一种**元数据**。**它主要用在编译期间对代码进行扫描或在运行期间通过反射执行相应的操作**。

## 注解

注解分类： 

- @RetentionPolicy：表示注解保留到哪个阶段。
- @Target：表示该注解可以用于什么地方。
- @Documented：表示将注解包含在 Javadoc 中。
- @Inherite：表示允许子类继承父类中的注解。

@RetentionPolicy中的参数说明：

- SOURCE：注解将被编译器丢弃。
- CLASS：注解在class文件中可用，但会被VM丢弃。
- RUNTIME：VM将在运行期间保留注解，因此可以通过反射机制读取注解的信息。

@RetentionPolicy中的ElementType参数值说明：

- TYPE：类、接口（包括注解类型）或enum声明
- FIELD：域声明（包括enum实例）
- METHOD：方法声明
- PARAMETER：参数声明
- CONSTRUCTOR：构造函数声明
- LOCAL_VARIABLE：局部变量声明
- ANNOTATION_TYPE：注解类型声明
- PACKAGE：包声明
- TYPE_PARAMETER：类型参数（在1.8中添加)
- TYPE_USE：类型使用（在1.8中添加)

注解具有的一般功能：

1. 根据代码里面标识的相应注解生成Javadoc。
2. 根据代码里面标识的相应注解在运行时执行相应的行为。
3. 根据代码里面标识的相应注解在编译时进行代码扫描检测。
4. 根据代码里面标识的相应注解在编译时生成Java类，文本文件等。

例子：

```
@Retention(RetentionPolicy.SOURCE)
@Target(ElementType.TYPE)
public @interface JavaDoc {

    String value();
}


@JavaDoc("这是用户数据类")
public class User {

    private int age;
    private String name;
}
```

`@JavaDoc` 注解用于修饰类，并且该注解会在编译器编译时被丢弃。

---

## 注解解析

注解解析主要通过反射和 `Processor` 。

### 运行时反射

运用 Java 提供的反射技术，可以在程序**运行时**获得类中所有相关注解的信息。

使用上面例子中的注解：

```
public class TestJavaDocAnnotation {

    public static void main(String[] args) {
        Class<User> clazz = User.class;
        JavaDoc javaDocAnnotation = clazz.getAnnotation(JavaDoc.class);
        System.out.println(javaDocAnnotation);
    }

}
输出结果：null
```

上面的输出结果为 null，这是因为 `@JavaDoc` 注解在编译器编译`User.java`文件时被丢弃。

修改`@JavaDoc` 注解 `@Retention(RetentionPolicy.SOURCE)` 为`@Retention(RetentionPolicy.RUNTIME)`，再重新运行，输出结果：

```
@com.wangjiang.example.annotation.JavaDoc(value="这是用户数据类")
```

注解被 VM 保留到了运行期。

当然，还可以通过反射获取类中相关字段、方法、方法参数等的注解信息。

### 解析器

通过编写注解解析器，在程序编译的时候，对注解标记的文件进行扫描。扫描可以发现程序错误或者生成相应文件（Java 类、HTML文档，txt文档等）。

编写注解解析器，主要通过实现类 `AbstractProcessor`来实现。

下面从创建Module，创建类 AbstractProcessor 的实现类，创建 javax.annotation.processing.Processor 文件 三方面 来说明怎样编写注解解析器。

#### 创建Module

在Android Studio 中创建注解解析的 Module 时，要选择成 **Java library**。

注：如果开发时，有些 Java 类找不到，需要在 Mudule 的 builde.gradle文件中添加：

```
dependencies {
    //...
    compileOnly files(org.gradle.internal.jvm.Jvm.current().getToolsJar())
}
```

当其它 Module 要引入使用这个 Module 的时候，需要在其它 Module 的 build.gradle 文件中添加依赖。比如这里 在项目主 Module 要使用，则在主 Module的 build.gradle 中添加依赖：

```
dependencies {
    // ...
    annotationProcessor project(path: ':注解解析 Module 的名字') //必须要添加这个
    implementation project(path: ':注解解析 Module 的名字')
}
```

如果编写的注解解析 Module 只是在编译时扫描，还可以将依赖修改为：

```
dependencies {
    // ...
    annotationProcessor project(path: ':注解解析 Module 的名字') //必须要添加这个
    complileOnly project(path: ':注解解析 Module 的名字')
}
```

#### 创建类 AbstractProcessor 的实现类

编写注解解析器，需要实现类 `AbstractProcessor`：

```
public class JavaDocProcessor extends AbstractProcessor {


    @Override
    public Set<String> getSupportedOptions() {
        //返回此注解解析器识别的选项
        return super.getSupportedOptions();
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        // 返回此注解解析器支持的注解类型的名称
        return super.getSupportedAnnotationTypes();
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        // 返回此注解解析器支持的最新的 JDK 版本
        return SourceVersion.latestSupported();
    }

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        //可以通过 processingEnv 获得当前解析器环境信息
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        //解析注解
        return false;
    }

}
```

`AbstractProcessor` 是一个抽象类，它实现了 `Processor` 接口。

`Processor` 接口的实现类必须提供一个公共的无参数构造方法，虚拟机工具将使用该构造方法实例化 `Processor`。

虚拟机工具框架与实现此接口的类交互过程：

1. 创建 `Processor` 对象 。
2. 调用 `Processor` 对象 的 `init` 方法。
3. 调用 `Processor` 对象 的 `getSupportedAnnotationTypes、getSupportedOptions 和 getSupportedSourceVersion` 方法。
4. 调用 `Processor` 对象 的调用 `process` 方法

如果想进一步了解接口`Processor` ，可查看 [Processor](http://tool.oschina.net/apidocs/apidoc?api=jdk-zh) 文档。

#### 创建 javax.annotation.processing.Processor 文件

除了实现类 `AbstractProcessor`，还需要在 main 文件夹下，与 java 同一级目录中创建 resources 等文件：
![在这里插入图片描述](/images/posts/2019-08-12-Android-Custom-Annotation/p1.png)
首先创建 resources 文件夹，再创建 META-INF/services/javax.annotation.processing.Processor 文件，在javax.annotation.processing.Processor 文件 中指明注解解析器类 `JavaDocProcessor` 的 类路径：`com.example.compiler.JavaDocProcessor`。

通过上面步骤，就可以开发注解解析器了，主要是在`com.example.compiler.JavaDocProcessor`类中写解析逻辑。

----

## 实例：生成 Java 文件

项目开发过程中会产生很多工具（util）类，新员工或不熟悉项目的人在开发新需求时，可能一时没有找到相应的工具（util）类或工具方法，就会去重复创建一个。如果能够提供一个可以找到所有工具类或工具方法的文档，那么就可以避免这种问题。

在项目中新建Module为：LibJavaDoc

![](/images/posts/2019-08-12-Android-Custom-Annotation/p1.png)



定义注解类`@JavaDoc`：

```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface JavaDoc {

}
```

注解解析类`JavaDocProcessor` ：

```
public class JavaDocProcessor extends AbstractProcessor {

    /**
     * 操作元素的工具方法
     */
    private Elements mElementUtil;
    /**
     * 用来创建新源、类或辅助文件的 Filer
     */
    private Filer mFiler;
    /**
     * 用来报告错误、警报和其他通知的 Messager
     */
    private Messager mMessager;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        mElementUtil = processingEnvironment.getElementUtils();
        mFiler = processingEnvironment.getFiler();
        mMessager = processingEnvironment.getMessager();

    }

    @Override
    public Set<String> getSupportedOptions() {
        return super.getSupportedOptions();
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        return Collections.singleton(JavaDoc.class.getCanonicalName());
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }


    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        if (!set.isEmpty()) {
            for (TypeElement typeElement : set) {
                Set<? extends Element> elements = roundEnvironment.getElementsAnnotatedWith(typeElement);//得到有标记@JavaDoc注解的元素
                for (Element element : elements) {
                    PackageElement pkgElement = (PackageElement) element.getEnclosingElement();//获得标记@JavaDoc注解包元素
                    List<JavaDocModel> javaDocModels = new ArrayList<>();
                    JavaDocModel classModel = getJavaDocModel(element, null);
                    if (classModel != null) {
                        javaDocModels.add(classModel);
                        String className = classModel.getName();//获得标记@JavaDoc注解的类名
                        List<? extends Element> enclosedElements = element.getEnclosedElements();//获得标记@JavaDoc注解的类中的所有方法元素
                        for (Element enclosedElement : enclosedElements) {
                            JavaDocModel methodModel = getJavaDocModel(enclosedElement, className);
                            if (methodModel != null)
                                javaDocModels.add(methodModel);
                        }
                    }
                    writeToFile(pkgElement.getQualifiedName().toString(), "JavaDoc", javaDocModels);
                }
            }

            return true;
        }
        return false;
    }

    /**
     * 将类或方法，注释写成 Java 文件
     * @param pkdName 生成的 Java 文件的包名
     * @param className 生成的 Java 文件的类名
     * @param javaDocModels 要生成的内容
     */
    private void writeToFile(String pkdName, String className, List<JavaDocModel> javaDocModels) {

        try {
            StringBuilder builder = new StringBuilder()
                    .append("package ").append(pkdName).append(";").append("\n")
                    .append("class JavaDoc").append("{").append("\n")
                    .append("\t").append("/**").append("\n");
            for (JavaDocModel javaDocModel : javaDocModels) {
                builder.append("\t").append("*");
                builder.append(javaDocModel.toString());
                builder.append("\n");
            }
            builder.append("\t").append("*/").append("\n")
                    .append("}");

            JavaFileObject fileObject = mFiler.createSourceFile(pkdName + "." + className);//生成 Java 文件
            Writer writer = fileObject.openWriter();
            writer.append(builder.toString());
            writer.flush();
            writer.close();

        } catch (IOException e) {
            e.printStackTrace();
        }

    }

    /**
     * 根据元素封装成数据类
     *
     * @param element   相应元素
     * @param className 类名
     * @return 封装成的数据类
     */
    private JavaDocModel getJavaDocModel(Element element, String className) {
        mMessager.printMessage(Diagnostic.Kind.NOTE, "element=" + element.toString());
        JavaDocModel javaDocModel = null;
        if (element.getKind() == ElementKind.CLASS || element.getKind() == ElementKind.METHOD) {
            javaDocModel = new JavaDocModel();
            String docComment = mElementUtil.getDocComment(element);//得到类或方法的注释
            if (docComment != null) {
                int index = docComment.indexOf("@");
                if (index != -1) {
                    docComment = docComment.substring(0, index).trim();
                } else {
                    docComment = docComment.trim();
                }
            }
            javaDocModel.setComment(docComment);
            if (element instanceof TypeElement) {
                javaDocModel.setName(element.toString());
            } else if (element instanceof ExecutableElement) {
                javaDocModel.setName(className + "#" + element.toString());
            }
        }
        return javaDocModel;
    }
}
```

对注解解析类`JavaDocProcessor` 中的一些类做下说明：

- Elements：操作元素的工具方法。

- Filer：用来创建新源、类或辅助文件。

- Messager：用来报告错误、警报和其他通知（在控制台打印日志）。

- Element：表示一个程序元素，比如包、类或者方法。每个元素都表示一个静态的语言级构造（不表示虚拟机的运行时构造）。
  
  关于这些类的更多API ，可以查看 [JDK 文档](http://tool.oschina.net/apidocs/apidoc?api=jdk-zh)。

这里还有一个非常有用的类`Trees`，想了解更多可以查看它的 [API 文档](https://docs.oracle.com/en/java/javase/11/docs/api/jdk.compiler/com/sun/source/util/Trees.html)。

```
 private Trees mTrees;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        mTrees = Trees.instance(processingEnvironment);
    }
```

要想更简单的生成 Java 类，可以使用第三方库 [javapoet](https://github.com/square/javapoet)。

数据类`JavaDocModel`：

```
public class JavaDocModel {

    /**
     * 名字
     */
    private String name;
    /**
     * 注释
     */
    private String comment;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getComment() {
        return comment;
    }

    public void setComment(String comment) {
        this.comment = comment;
    }

    @Override
    public String toString() {
        return "{" +
                "{@link " + name + '}' +
                ", " + comment +
                '}';
    }
}
```

### 测试

在一个工具类`@DisplayUtil`中添加注解`@JavaDoc` :

```
package com.example.test.util;
/**
 * 页面相关
 * @author wangjiang wangjiang7747@gmail.com
 * @version V1.0
 */
@JavaDoc
public final class DisplayUtil {

    private DisplayUtil() {
    }

    /**
     * 获得StatusBar的高度
     *
     * @param context 上下文对象
     * @return 状态栏的高度
     */
    public static int getStatusBarHeight(Context context) {
        Resources resources = context.getResources();
        int resourceId = resources.getIdentifier("status_bar_height", "dimen",
                "android");
        int statusBarHeight = resources.getDimensionPixelSize(resourceId);
        return statusBarHeight;
    }

    ...省略部分代码
}
```

 在项目中通过`./gradlew build` 命令 构建项目生成类 `com.example.test.util.JavaDoc`  ，JavaDoc.java 文件存在于主 Module 的 /build/generated/source/apt/debug/com/example/test/util/ 文件夹下面
![在这里插入图片描述](/images/posts/2019-08-12-Android-Custom-Annotation/p2.png)
com/example/test/util 是获取到的 DisplayUtil.java 类的包名，在这里生成 JavaDoc.java 类 与 DisplayUtil.java 类在同一个包下。所以在JavaDoc.java 类中，可以直接通过快捷键跳转到DisplayUtil.java 类中。另外，生成的该类，也可以在运行时通过反射获取到。

## 总结

1. 在定义注解时需要明确注解修饰符的含义：`@RetentionPolicy，@Target， @Target，@Inherite`。

2. 注解解析可以通过**运行时反射**和 **编译时扫描**。

3. 定义注解解析器需要注意三个方面：**创建 Module 要选择成 Java library**，**实现类AbstractProcessor**，**创建 javax.annotation.processing.Processor 文件**。

4. 在注解解析类`AbstractProcessor`的实现类中，可以用类 `Elements，Filer，Messager，Trees`进行帮助注解解析。主要不了解这些类的API的用法，可以查看 [JDK 文档](http://tool.oschina.net/apidocs/apidoc?api=jdk-zh)。
