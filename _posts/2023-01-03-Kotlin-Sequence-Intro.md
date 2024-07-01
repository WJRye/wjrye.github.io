---
layout: post
title: Kotlin 惰性集合操作-序列 Sequence
categories: [Kotlin]
description: Kotlin Sequence 是为了减少链式操作变换的总次数。
keywords: Kotlin, Sequence
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
topmost: false
---

## 集合操作函数 和 序列

在了解 Kotlin 惰性集合之前，先看一下 Koltin 标准库中的一些集合操作函数。

定义一个数据模型 Person 和 Book 类：

```
data class Person(val name: String, val age: Int) 

data class Book(val title: String, val authors: List<String>)

```

filter 和 map 操作：

```
    val people = listOf<Person>(
            Person("xiaowang", 30),
            Person("xiaozhang", 32),
            Person("xiaoli", 28)
        )
        //大于 30 岁的人的名字集合列表
     people.filter { it.age >= 30 }.map(Person::name)
```

count 操作：

```
   val people = listOf<Person>(
            Person("xiaowang", 30),
            Person("xiaozhang", 32),
            Person("xiaoli", 28)
        )
        //小于 30 岁人的个数
   people.count { it.age < 30 }
```
flatmap 操作：

```
      val books = listOf<Book>(
            Book("Java 语言程序设计", arrayListOf("xiaowang", "xiaozhang")),
            Book("Kotlin 语言程序设计", arrayListOf("xiaoli", "xiaomao")),
        )
        // 所有书的名字集合列表
        books.flatMap { it.authors }.toList()
```
在上面这些函数，**每做一步操作，都会创建中间集合，也就是每一步的中间结果都被临时存储在一个临时集合中**。

filter 函数源码：

```
public inline fun <T> Iterable<T>.filter(predicate: (T) -> Boolean): List<T> {
    //创建一个新的集合列表
    return filterTo(ArrayList<T>(), predicate)
}

public inline fun <T, C : MutableCollection<in T>> Iterable<T>.filterTo(destination: C, predicate: (T) -> Boolean): C {
    for (element in this) if (predicate(element)) destination.add(element)
    return destination
}
```

map 函数源码：

```
public inline fun <T, R> Iterable<T>.map(transform: (T) -> R): List<R> {
    //创建一个新的集合列表
    return mapTo(ArrayList<R>(collectionSizeOrDefault(10)), transform)
}

public inline fun <T, R, C : MutableCollection<in R>> Iterable<T>.mapTo(destination: C, transform: (T) -> R): C {
    for (item in this)
        destination.add(transform(item))
    return destination
}

```
如果被操作的元素过多，假设 people 或 books 超过 50个、100个，那么 函数链式调用 `如：fliter{}.map{}` 就会变得低效，且浪费内存。

Kotlin 为解决上面这种问题，提供了惰性集合操作 `Sequence` 接口。这个接口表示一个可以逐个列举的元素列表。**Sequence 只提供了一个 方法， iterator，用来从序列中获取值。**

```
public interface Sequence<out T> {
    /**
     * Returns an [Iterator] that returns the values from the sequence.
     *
     * Throws an exception if the sequence is constrained to be iterated once and `iterator` is invoked the second time.
     */
    public operator fun iterator(): Iterator<T>
}

public inline fun <T> Sequence(crossinline iterator: () -> Iterator<T>): Sequence<T> = object : Sequence<T> {
    override fun iterator(): Iterator<T> = iterator()
}

/**
 * Creates a sequence that returns all elements from this iterator. The sequence is constrained to be iterated only once.
 *
 * @sample samples.collections.Sequences.Building.sequenceFromIterator
 */
public fun <T> Iterator<T>.asSequence(): Sequence<T> = Sequence { this }.constrainOnce()
```
**序列中的元素求值是惰性的。因此，可以使用序列更高效地对集合元素执行链式操作，而不需要创建额外的集合来保存过程中产生的中间结果**。关于这个**惰性**是怎么来的，后面再详细解释。

可以调用扩展函数 asSequence 把任意集合转换成序列，调用 toList 来做反向的转换。

```
 val people = listOf<Person>(
            Person("xiaowang", 30),
            Person("xiaozhang", 32),
            Person("xiaoli", 28)
        )
  people.asSequence().filter { it.age >= 30 }.map(Person::name).toList()
```

```
 val books = listOf<Book>(
            Book("Java 语言程序设计", arrayListOf("xiaowang", "xiaozhang")),
            Book("Kotlin 语言程序设计", arrayListOf("xiaoli", "xiaomao")),
        )
 books.asSequence().flatMap { it.authors }.toList()
```

## 序列中间和末端操作
序列操作分为两类：中间的和末端的。一次中间操作返回的是另一个序列，这个新序列知道如何变换原始序列中的元素。而一次末端返回的是一个**结果**，这个结果可能是集合、元素、数字，或者其他从初始集合的变换序列中获取的任意对象。

![](/images/posts/2023-01-03-Kotlin-Sequence-Intro/p1.png)

中间操作始终是**惰性**的。

下面从例子来理解这个**惰性**：

```
listOf(1, 2, 3, 4).asSequence().map {
            println("map${it}")
            it * it
        }.filter {
            println("filter${it}")
            it % 2 == 0
        }
```
上面这段代码在控制台不会输出任何内容（因为没有末端操作）。

```
listOf(1, 2, 3, 4).asSequence().map {
            println("map${it}")
            it * it
        }.filter {
            println("filter${it}")
            it % 2 == 0
        }.toList()
 
 控制台输出：
D/TestSequence: map1
D/TestSequence: filter1
D/TestSequence: map2
D/TestSequence: filter4
D/TestSequence: map3
D/TestSequence: filter9
D/TestSequence: map4
D/TestSequence: filter16
```
在末端操作 `.toList()`的时候，`map` 和 `filter` 变换才被执行，而且元素是被逐个执行的。并不是所有元素经在 map 操作执行完成后，再执行 filter 操作。

为什么元素是逐个被执行，首先看下 `toList()` 方法：

```
public fun <T> Sequence<T>.toList(): List<T> {
    return this.toMutableList().optimizeReadOnlyList()
}

public fun <T> Sequence<T>.toMutableList(): MutableList<T> {
    return toCollection(ArrayList<T>())
}

public fun <T, C : MutableCollection<in T>> Sequence<T>.toCollection(destination: C): C {
    for (item in this) {
        destination.add(item)
    }
    return destination
}
```
最后的 `toCollection` 方法中的 ` for (item in this)`，其实就是调用 `Sequence`  中的迭代器 `Iterator` 进行元素迭代。其中这个 `this` 来自于 `filter`，也就是使用  `filter` 的 `Iterator` 进行元素迭代。来看下 `filter` ：

```
public fun <T> Sequence<T>.filter(predicate: (T) -> Boolean): Sequence<T> {
    return FilteringSequence(this, true, predicate)
}

internal class FilteringSequence<T>(
    private val sequence: Sequence<T>,
    private val sendWhen: Boolean = true,
    private val predicate: (T) -> Boolean
) : Sequence<T> {

    override fun iterator(): Iterator<T> = object : Iterator<T> {
        val iterator = sequence.iterator()
        var nextState: Int = -1 // -1 for unknown, 0 for done, 1 for continue
        var nextItem: T? = null

        private fun calcNext() {
            while (iterator.hasNext()) {
                val item = iterator.next()
                if (predicate(item) == sendWhen) {
                    nextItem = item
                    nextState = 1
                    return
                }
            }
            nextState = 0
        }

        override fun next(): T {
            if (nextState == -1)
                calcNext()
            if (nextState == 0)
                throw NoSuchElementException()
            val result = nextItem
            nextItem = null
            nextState = -1
            @Suppress("UNCHECKED_CAST")
            return result as T
        }

        override fun hasNext(): Boolean {
            if (nextState == -1)
                calcNext()
            return nextState == 1
        }
    }
}
```
`filter` 中又会使用上一个 `Sequence`  的 `sequence.iterator()` 进行元素迭代。再看下 `map` ：

```
public fun <T, R> Sequence<T>.map(transform: (T) -> R): Sequence<R> {
    return TransformingSequence(this, transform)
}

internal class TransformingSequence<T, R>
constructor(private val sequence: Sequence<T>, private val transformer: (T) -> R) : Sequence<R> {
    override fun iterator(): Iterator<R> = object : Iterator<R> {
        val iterator = sequence.iterator()
        override fun next(): R {
            return transformer(iterator.next())
        }

        override fun hasNext(): Boolean {
            return iterator.hasNext()
        }
    }

    internal fun <E> flatten(iterator: (R) -> Iterator<E>): Sequence<E> {
        return FlatteningSequence<T, R, E>(sequence, transformer, iterator)
    }
}
```
也是使用上一个 `Sequence`  的 `sequence.iterator()` 进行元素迭代。所以以此类推，最终会使用转换为 `asSequence()`  的源 `iterator()`。

下面自定义一个 `Sequence`   来验证上面的猜想：

```
listOf(1, 2, 3, 4).asSequence().mapToString {
            Log.d("TestSequence","mapToString${it}")
            it.toString()
        }.toList()

    fun <T> Sequence<T>.mapToString(transform: (T) -> String): Sequence<String> {
        return TransformingStringSequence(this, transform)
    }

    class TransformingStringSequence<T>
    constructor(private val sequence: Sequence<T>, private val transformer: (T) -> String) : Sequence<String> {
        override fun iterator(): Iterator<String> = object : Iterator<String> {
            val iterator = sequence.iterator()
            override fun next(): String {
                val next = iterator.next()
                Log.d("TestSequence","next:${next}")
                return transformer(next)
            }

            override fun hasNext(): Boolean {
                return iterator.hasNext()
            }
        }
    }

控制台输出：
D/TestSequence: next:1
D/TestSequence: mapToString1
D/TestSequence: next:2
D/TestSequence: mapToString2
D/TestSequence: next:3
D/TestSequence: mapToString3
D/TestSequence: next:4
D/TestSequence: mapToString4
```
所以这就是 `Sequence` 为什么在获取结果的时候才会被应用，也就是末端操作被调用的时候，才会依次处理每个元素，这也是 被称为**惰性集合操作的原因**。

经过一系列的 序列操作，每个元素逐个被处理，那么优先处理 `filter` 序列，其实可以减少变换的总次数。因为每个序列都是使用上一个序列的 `sequence.iterator()` 进行元素迭代。

## 创建序列
在集合操作上，可以使用集合直接调用 `asSequence()` 转换为序列。那么不是集合，有类似集合一样的变换，该怎么操作呢。

下面以求 1到100 的所有自然数之和为例子：

```
val naturalNumbers = generateSequence(0) { it + 1 }
val numbersTo100 = naturalNumbers.takeWhile { it <= 100 }
val sum = numbersTo100.sum()
println(sum)
控制台输出：
5050
```
先看下 `generateSequence` 源码：

```
public fun <T : Any> generateSequence(seed: T?, nextFunction: (T) -> T?): Sequence<T> =
    if (seed == null)
        EmptySequence
    else
        GeneratorSequence({ seed }, nextFunction)

private class GeneratorSequence<T : Any>(private val getInitialValue: () -> T?, private val getNextValue: (T) -> T?) : Sequence<T> {
    override fun iterator(): Iterator<T> = object : Iterator<T> {
        var nextItem: T? = null
        var nextState: Int = -2 // -2 for initial unknown, -1 for next unknown, 0 for done, 1 for continue

        private fun calcNext() {
            //getInitialValue 获取的到就是 generateSequence 的第一个参数 0
            //getNextValue 获取到的就是 generateSequence 的第二个参数 it+1，这个it 就是 nextItem!!
            nextItem = if (nextState == -2) getInitialValue() else getNextValue(nextItem!!)
            nextState = if (nextItem == null) 0 else 1
        }

        override fun next(): T {
            if (nextState < 0)
                calcNext()

            if (nextState == 0)
                throw NoSuchElementException()
            val result = nextItem as T
            // Do not clean nextItem (to avoid keeping reference on yielded instance) -- need to keep state for getNextValue
            nextState = -1
            return result
        }

        override fun hasNext(): Boolean {
            if (nextState < 0)
                calcNext()
            return nextState == 1
        }
    }
}
```
上面代码其实就是创建一个 `Sequence` 接口实现类，并实现它的 `iterator` 接口方法，返回一个 `Iterator` 迭代器。

```
public fun <T> Sequence<T>.takeWhile(predicate: (T) -> Boolean): Sequence<T> {
    return TakeWhileSequence(this, predicate)
}

internal class TakeWhileSequence<T>
constructor(
    private val sequence: Sequence<T>,
    private val predicate: (T) -> Boolean
) : Sequence<T> {
    override fun iterator(): Iterator<T> = object : Iterator<T> {
        val iterator = sequence.iterator()
        var nextState: Int = -1 // -1 for unknown, 0 for done, 1 for continue
        var nextItem: T? = null

        private fun calcNext() {
            if (iterator.hasNext()) {
                //iterator.next() 调用的就是上一个 GeneratorSequence 的 next 方法，而返回值就是它的 it+1
                val item = iterator.next()
                //判断条件，也就是 it <= 100 -> item <= 100
                if (predicate(item)) {
                    nextState = 1
                    nextItem = item
                    return
                }
            }
            nextState = 0
        }

        override fun next(): T {
            if (nextState == -1)
                calcNext() // will change nextState
            if (nextState == 0)
                throw NoSuchElementException()
            @Suppress("UNCHECKED_CAST")
            val result = nextItem as T

            // Clean next to avoid keeping reference on yielded instance
            nextItem = null
            nextState = -1
            return result
        }

        override fun hasNext(): Boolean {
            if (nextState == -1)
                calcNext() // will change nextState
            return nextState == 1
        }
    }
}
```
在 `TakeWhileSequence` 的 `next` 方法中，会优先调用内部方法 `calcNext`，而这个方法内部又是调用 `GeneratorSequence`的 `next`方法，这样就 拿到了当前值 it+1（上一个是0+1，下一个就是1+1），拿到值后再判断 ` it <= 100 -> item <= 100`。

```
public fun Sequence<Int>.sum(): Int {
    var sum: Int = 0
    for (element in this) {
        sum += element
    }
    return sum
}
```
`sum` 方法是序列的末端操作，也就是获取结果。`for (element in this) `，调用上一个 `Sequence`  中的迭代器 `Iterator` 进行元素迭代，以此类推，直到调用 源 `Sequence`  中的迭代器 `Iterator` 进行元素迭代。

## 总结
Kotlin 标准库提供的集合操作函数：filter，map, flatmap 等，在操作的时候会创建存储中间结果的临时列表，当集合元素较多时，这种链式操作就会变得低效。为了解决这种问题，Kotlin 提供了惰性集合操作 `Sequence` 接口，只有在 末端操作被调用的时候，也就是获取结果的时候，序列中的元素才会被逐个执行，处理完第一个元素后，才会处理第二个元素，这样中间操作是被延期执行的。而且因为是顺序地去执行每一个元素，所以可以先做 filter 变换，再做 map 变换，这样有助于减少变换的总次数。


