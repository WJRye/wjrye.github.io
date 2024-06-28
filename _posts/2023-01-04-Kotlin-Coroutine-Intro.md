---
layout: post
title: Kotlin 协程探索
categories: [Kotlin]
description: 协程是 Kotlin 提供的一套线程 API 框架，可以很方便的做线程切换。
keywords: Kotlin, Coroutine
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
topmost: false
---

## Kotlin 协程是什么？

本文只是自己经过研究后，对 Kotlin 协程的理解概括，如有偏差，还请斧正。

简要概括：

> 协程是 Kotlin 提供的一套线程 API 框架，可以很方便的做线程切换。 而且在不用关心线程调度的情况下，能轻松的做并发编程。也可以说协程就是一种并发设计模式。

下面是使用传统线程和协程执行任务：

```
       Thread{
            //执行耗时任务
        }.start()

        val executors = Executors.newCachedThreadPool()
        executors.execute {
          //执行耗时任务
        }

       GlobalScope.launch(Dispatchers.IO) {
          //执行耗时任务
        }
```

在实际应用开发中，通常是在主线中去启动子线程执行耗时任务，等耗时任务执行完成，再将结果给主线程，然后刷新UI：

```
       Thread{
            //执行耗时任务
            runOnMainThread { 
                //获取耗时任务结果，刷新UI
            }
        }.start()

        val executors = Executors.newCachedThreadPool()
        executors.execute {
            //执行耗时任务
            runOnMainThread {
                //获取耗时任务结果，刷新UI
            }
        }

        Observable.unsafeCreate<Unit> {
            //执行耗时任务
        }.subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread()).subscribe {
            //获取耗时任务结果，刷新UI
        }

        GlobalScope.launch(Dispatchers.Main) {
            val result = withContext(Dispatchers.IO){
                //执行耗时任务
            }
            //直接拿到耗时任务结果，刷新UI
            refreshUI(result)
        }
```

从上面可以看到，使用Java 的 `Thread` 和 `Executors` 都需要手动去处理线程切换，这样的代码不仅不优雅，而且有一个重要问题，那就是要去处理与生命周期相关的上下文判断，这导致逻辑变复杂，而且容易出错。

RxJava 是一套优雅的异步处理框架，代码逻辑简化，可读性和可维护性都很高，很好的帮我们处理线程切换操作。这在 Java 语言环境开发下，是如虎添翼，但是在 Kotlin 语言环境中开发，如今的协程就比 RxJava 更方便，或者说更有优势。

下面看一个 Kotlin 中使用协程的例子：

```
        GlobalScope.launch(Dispatchers.Main) {
            Log.d("TestCoroutine", "launch start: ${Thread.currentThread()}")
            val numbersTo50Sum = withContext(Dispatchers.IO) {
                //在子线程中执行 1-50 的自然数和
                Log.d("TestCoroutine", "launch:numbersTo50Sum: ${Thread.currentThread()}")
                delay(1000)
                val naturalNumbers = generateSequence(0) { it + 1 }
                val numbersTo50 = naturalNumbers.takeWhile { it <= 50 }
                numbersTo50.sum()
            }

            val numbers50To100Sum = withContext(Dispatchers.IO) {
               //在子线程中执行 51-100 的自然数和
                Log.d("TestCoroutine", "launch:numbers50To100Sum: ${Thread.currentThread()}")
                delay(1000)
                val naturalNumbers = generateSequence(51) { it + 1 }
                val numbers50To100 = naturalNumbers.takeWhile { it in 51..100 }
                numbers50To100.sum()
            }

            val result = numbersTo50Sum + numbers50To100Sum
            Log.d("TestCoroutine", "launch end:result=$result ${Thread.currentThread()}")
        }
        Log.d("TestCoroutine", "Hello World!,${Thread.currentThread()}")
控制台输出结果：
D/TestCoroutine: Hello World!,Thread[main,5,main]
D/TestCoroutine: launch start: Thread[main,5,main]
D/TestCoroutine: launch:numbersTo50Sum: Thread[DefaultDispatcher-worker-1,5,main]
D/TestCoroutine: launch:numbers50To100Sum: Thread[DefaultDispatcher-worker-1,5,main]
D/TestCoroutine: launch end:result=5050 Thread[main,5,main]
```

在上面的代码中：

- `launch` 是一个函数，用于创建协程并将其函数主体的执行分派给相应的调度程序。
- `Dispatchers.MAIN` 指示此协程应在为 UI 操作预留的主线程上执行。
- `Dispatchers.IO` 指示此协程应在为 I/O 操作预留的线程上执行。
- `withContext(Dispatchers.IO)` 将协程的执行操作移至一个 I/O 线程。

从控制台输出结果中，可以看出在计算 1-50 和 51-100 的自然数和的时候，线程是从主线程（`Thread[main,5,main]`）切换到了协程的线程（`DefaultDispatcher-worker-1,5,main`），这里计算 1-50 和 51-100 都是同一个子线程。

在这里有一个重要的现象，代码从逻辑上看起来是同步的，并且启动协程执行任务的时候，没有阻塞主线程继续执行相关操作，而且在协程中的异步任务执行完成之后，又自动切回了主线程。这就是 Kotlin 协程给开发做并发编程带来的好处。这也是有个概念的来源： Kotlin 协程**同步非阻塞**。

**同步非阻塞**”是真的“**同步非阻塞**” 吗？下面探究一下其中的猫腻，通过 Android Studio ，查看 .class 文件中的上面一段代码：

```
      BuildersKt.launch$default((CoroutineScope)GlobalScope.INSTANCE, (CoroutineContext)Dispatchers.getMain(), (CoroutineStart)null, (Function2)(new Function2((Continuation)null) {
         int I$0;
         int label;

         @Nullable
         public final Object invokeSuspend(@NotNull Object $result) {
            Object var10000;
            int numbersTo50Sum;
            label17: {
               Object var5 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
               Function2 var10001;
               CoroutineContext var6;
               switch(this.label) {
               case 0:
                  ResultKt.throwOnFailure($result);
                  Log.d("TestCoroutine", "launch start: " + Thread.currentThread());
                  var6 = (CoroutineContext)Dispatchers.getIO();
                  var10001 = (Function2)(new Function2((Continuation)null) {
                     int label;

                     @Nullable
                     public final Object invokeSuspend(@NotNull Object $result) {
                        Object var4 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
                        switch(this.label) {
                        case 0:
                           ResultKt.throwOnFailure($result);
                           Log.d("TestCoroutine", "launch:numbersTo50Sum: " + Thread.currentThread());
                           this.label = 1;
                           if (DelayKt.delay(1000L, this) == var4) {
                              return var4;
                           }
                           break;
                        case 1:
                           ResultKt.throwOnFailure($result);
                           break;
                        default:
                           throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
                        }

                        Sequence naturalNumbers = SequencesKt.generateSequence(Boxing.boxInt(0), (Function1)null.INSTANCE);
                        Sequence numbersTo50 = SequencesKt.takeWhile(naturalNumbers, (Function1)null.INSTANCE);
                        return Boxing.boxInt(SequencesKt.sumOfInt(numbersTo50));
                     }

                     @NotNull
                     public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {
                        Intrinsics.checkNotNullParameter(completion, "completion");
                        Function2 var3 = new <anonymous constructor>(completion);
                        return var3;
                     }

                     public final Object invoke(Object var1, Object var2) {
                        return ((<undefinedtype>)this.create(var1, (Continuation)var2)).invokeSuspend(Unit.INSTANCE);
                     }
                  });
                  this.label = 1;
                  var10000 = BuildersKt.withContext(var6, var10001, this);
                  if (var10000 == var5) {
                     return var5;
                  }
                  break;
               case 1:
                  ResultKt.throwOnFailure($result);
                  var10000 = $result;
                  break;
               case 2:
                  numbersTo50Sum = this.I$0;
                  ResultKt.throwOnFailure($result);
                  var10000 = $result;
                  break label17;
               default:
                  throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
               }

               numbersTo50Sum = ((Number)var10000).intValue();
               var6 = (CoroutineContext)Dispatchers.getIO();
               var10001 = (Function2)(new Function2((Continuation)null) {
                  int label;

                  @Nullable
                  public final Object invokeSuspend(@NotNull Object $result) {
                     Object var4 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
                     switch(this.label) {
                     case 0:
                        ResultKt.throwOnFailure($result);
                        Log.d("TestCoroutine", "launch:numbers50To100Sum: " + Thread.currentThread());
                        this.label = 1;
                        if (DelayKt.delay(1000L, this) == var4) {
                           return var4;
                        }
                        break;
                     case 1:
                        ResultKt.throwOnFailure($result);
                        break;
                     default:
                        throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
                     }

                     Sequence naturalNumbers = SequencesKt.generateSequence(Boxing.boxInt(51), (Function1)null.INSTANCE);
                     Sequence numbers50To100 = SequencesKt.takeWhile(naturalNumbers, (Function1)null.INSTANCE);
                     return Boxing.boxInt(SequencesKt.sumOfInt(numbers50To100));
                  }

                  @NotNull
                  public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {
                     Intrinsics.checkNotNullParameter(completion, "completion");
                     Function2 var3 = new <anonymous constructor>(completion);
                     return var3;
                  }

                  public final Object invoke(Object var1, Object var2) {
                     return ((<undefinedtype>)this.create(var1, (Continuation)var2)).invokeSuspend(Unit.INSTANCE);
                  }
               });
               this.I$0 = numbersTo50Sum;
               this.label = 2;
               var10000 = BuildersKt.withContext(var6, var10001, this);
               if (var10000 == var5) {
                  return var5;
               }
            }

            int numbers50To100Sum = ((Number)var10000).intValue();
            int result = numbersTo50Sum + numbers50To100Sum;
            Log.d("TestCoroutine", "launch end:result=" + result + ' ' + Thread.currentThread());
            return Unit.INSTANCE;
         }

         @NotNull
         public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {
            Intrinsics.checkNotNullParameter(completion, "completion");
            Function2 var3 = new <anonymous constructor>(completion);
            return var3;
         }

         public final Object invoke(Object var1, Object var2) {
            return ((<undefinedtype>)this.create(var1, (Continuation)var2)).invokeSuspend(Unit.INSTANCE);
         }
      }), 2, (Object)null);
      Log.d("TestCoroutine", "Hello World!," + Thread.currentThread());
```

虽然上面 .class 文件中的代码比较复杂，但是从大体逻辑可以看出，Kotlin 协程也是通过回调接口来实现异步操作的，这也解释了 Kotlin 协程只是让代码逻辑是同步非阻塞，但是实际上并没有，只是 Kotlin 编译器为代码做了很多事情，这也是说 Kotlin 协程其实就是一套线程 API 框架的原因。

再看一个上面例子的变种：

```
        GlobalScope.launch(Dispatchers.Main) {
            Log.d("TestCoroutine", "launch start: ${Thread.currentThread()}")
            val numbersTo50Sum = async {
                withContext(Dispatchers.IO) {
                    Log.d("TestCoroutine", "launch:numbersTo50Sum: ${Thread.currentThread()}")
                    delay(2000)
                    val naturalNumbers = generateSequence(0) { it + 1 }
                    val numbersTo50 = naturalNumbers.takeWhile { it <= 50 }
                    numbersTo50.sum()
                }
            }

            val numbers50To100Sum = async {
                withContext(Dispatchers.IO) {
                    Log.d("TestCoroutine", "launch:numbers50To100Sum: ${Thread.currentThread()}")
                    delay(500)
                    val naturalNumbers = generateSequence(51) { it + 1 }
                    val numbers50To100 = naturalNumbers.takeWhile { it in 51..100 }
                    numbers50To100.sum()
                }
            }
            // 计算 1-50 和 51-100 的自然数和是两个并发操作
            val result = numbersTo50Sum.await() + numbers50To100Sum.await()
            Log.d("TestCoroutine", "launch end:result=$result ${Thread.currentThread()}")
        }
        Log.d("TestCoroutine", "Hello World!,${Thread.currentThread()}")

  控制台输出结果：
D/TestCoroutine: Hello World!,Thread[main,5,main]
D/TestCoroutine: launch start: Thread[main,5,main]
D/TestCoroutine: launch:numbersTo50Sum: Thread[DefaultDispatcher-worker-2,5,main]
D/TestCoroutine: launch:numbers50To100Sum: Thread[DefaultDispatcher-worker-1,5,main]
D/TestCoroutine: launch end:result=5050 Thread[main,5,main]
```

`async` 创建了一个协程，它让计算 1-50 和 51-100 的自然数和是两个**并发操作**。上面控制台输出结果可以看到计算 1-50  的自然数和是在线程 `Thread[DefaultDispatcher-worker-2,5,main]` 中，而计算 51-100  的自然数和是在另一个线程`Thread[DefaultDispatcher-worker-1,5,main]`中。

从上面的例子，协程在异步操作，也就是线程切换上：主线程启动子线程执行耗时操作，耗时操作执行完成将结果更新到主线程的过程中，代码逻辑简化，可读性高。

## suspend 是什么？

suspend 直译就是：挂起

suspend 是 Kotlin 语言中一个 关键字，用于修饰方法，当修饰方法时，表示这个方法只能被 suspend 修饰的方法调用或者在协程中被调用。

下面看一下将上面代码案例拆分成几个 suspend 方法：

```
    fun getNumbersTo100Sum() {
        GlobalScope.launch(Dispatchers.Main) {
            Log.d("TestCoroutine", "launch start: ${Thread.currentThread()}")
            val result = calcNumbers1To100Sum()
            Log.d("TestCoroutine", "launch end:result=$result ${Thread.currentThread()}")
        }
        Log.d("TestCoroutine", "Hello World!,${Thread.currentThread()}")
    }

    private suspend fun calcNumbers1To100Sum(): Int {
        return calcNumbersTo50Sum() + calcNumbers50To100Sum()
    }

    private suspend fun calcNumbersTo50Sum(): Int {
        return withContext(Dispatchers.IO) {
            Log.d("TestCoroutine", "launch:numbersTo50Sum: ${Thread.currentThread()}")
            delay(1000)
            val naturalNumbers = generateSequence(0) { it + 1 }
            val numbersTo50 = naturalNumbers.takeWhile { it <= 50 }
            numbersTo50.sum()
        }
    }

    private suspend fun calcNumbers50To100Sum(): Int {
        return withContext(Dispatchers.IO) {
            Log.d("TestCoroutine", "launch:numbers50To100Sum: ${Thread.currentThread()}")
            delay(1000)
            val naturalNumbers = generateSequence(51) { it + 1 }
            val numbers50To100 = naturalNumbers.takeWhile { it in 51..100 }
            numbers50To100.sum()
        }
    }
控制台输出结果：
D/TestCoroutine: Hello World!,Thread[main,5,main]
D/TestCoroutine: launch start: Thread[main,5,main]
D/TestCoroutine: launch:numbersTo50Sum: Thread[DefaultDispatcher-worker-3,5,main]
D/TestCoroutine: launch:numbers50To100Sum: Thread[DefaultDispatcher-worker-1,5,main]
D/TestCoroutine: launch end:result=5050 Thread[main,5,main]
```

**suspend 关键字标记方法时，其实是告诉 Kotlin 从协程内调用方法。所以这个“挂起”，并不是说方法或函数被挂起，也不是说线程被挂起**。

假设一个非 suspend 修饰的方法调用 suspend 修饰的方法会怎么样呢？

```
  private fun calcNumbersTo100Sum(): Int {
        return calcNumbersTo50Sum() + calcNumbers50To100Sum()
    }
```

此时，编译器会提示：

```
Suspend function 'calcNumbersTo50Sum' should be called only from a coroutine or another suspend function
Suspend function 'calcNumbers50To100' should be called only from a coroutine or another suspend function
```

下面查看 .class 文件中的上面方法 calcNumbers50To100Sum 代码：

```
   private final Object calcNumbers50To100Sum(Continuation $completion) {
      return BuildersKt.withContext((CoroutineContext)Dispatchers.getIO(), (Function2)(new Function2((Continuation)null) {
         int label;

         @Nullable
         public final Object invokeSuspend(@NotNull Object $result) {
            Object var4 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
            switch(this.label) {
            case 0:
               ResultKt.throwOnFailure($result);
               Log.d("TestCoroutine", "launch:numbers50To100Sum: " + Thread.currentThread());
               this.label = 1;
               if (DelayKt.delay(1000L, this) == var4) {
                  return var4;
               }
               break;
            case 1:
               ResultKt.throwOnFailure($result);
               break;
            default:
               throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
            }

            Sequence naturalNumbers = SequencesKt.generateSequence(Boxing.boxInt(51), (Function1)null.INSTANCE);
            Sequence numbers50To100 = SequencesKt.takeWhile(naturalNumbers, (Function1)null.INSTANCE);
            return Boxing.boxInt(SequencesKt.sumOfInt(numbers50To100));
         }

         @NotNull
         public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {
            Intrinsics.checkNotNullParameter(completion, "completion");
            Function2 var3 = new <anonymous constructor>(completion);
            return var3;
         }

         public final Object invoke(Object var1, Object var2) {
            return ((<undefinedtype>)this.create(var1, (Continuation)var2)).invokeSuspend(Unit.INSTANCE);
         }
      }), $completion);
   }
```

可以看到 `private suspend fun calcNumbers50To100Sum()` 经过 Kotlin 编译器编译后变成了`private final Object calcNumbers50To100Sum(Continuation $completion)`， `suspend` 消失了，方法多了一个参数 `Continuation $completion`，所以 `suspend`修饰 Kotlin 的方法或函数，编译器会对此方法做特殊处理。

另外，`suspend` 修饰的方法，也预示着这个方法是**耗时方法**，告诉方法调用者要使用协程。当执行 `suspend` 方法，也预示着要切换线程，此时主线程依然可以继续执行，而协程里面的代码可能被挂起了。

下面再稍为修改 `calcNumbers50To100Sum` 方法：

```
   private suspend fun calcNumbers50To100Sum(): Int {
        Log.d("TestCoroutine", "launch:numbers50To100Sum:start: ${Thread.currentThread()}")
        val sum= withContext(Dispatchers.Main) {
            Log.d("TestCoroutine", "launch:numbers50To100Sum: ${Thread.currentThread()}")
            delay(1000)
            val naturalNumbers = generateSequence(51) { it + 1 }
            val numbers50To100 = naturalNumbers.takeWhile { it in 51..100 }
            numbers50To100.sum()
        }
        Log.d("TestCoroutine", "launch:numbers50To100Sum:end: ${Thread.currentThread()}")
        return sum
    }
控制台输出结果：
D/TestCoroutine: Hello World!,Thread[main,5,main]
D/TestCoroutine: launch start: Thread[main,5,main]
D/TestCoroutine: launch:numbersTo50Sum: Thread[DefaultDispatcher-worker-3,5,main]
D/TestCoroutine: launch:numbers50To100Sum:start: Thread[main,5,main]
D/TestCoroutine: launch:numbers50To100Sum: Thread[main,5,main]
D/TestCoroutine: launch:numbers50To100Sum:end: Thread[main,5,main]
D/TestCoroutine: launch end:result=5050 Thread[main,5,main]
```

主线程不受协程线程的影响。

## 总结

Kotlin 协程是一套线程 API 框架，在 Kotlin 语言环境下使用它做并发编程比传统 Thread, Executors 和 RxJava 更有优势，代码逻辑上“同步非阻塞“，而且简洁，易阅读和维护。

`suspend` 是 Kotlin 语言中一个关键字，用于修饰方法，当修饰方法时，该方法只能被 `suspend` 修饰的方法和协程调用。此时，也预示着该方法是一个耗时方法，告诉调用者需要在协程中使用。

参考文档：

1. [Android 上的 Kotlin 协程](https://developer.android.google.cn/kotlin/coroutines?hl=zh_cn)
2. [Coroutines guide](https://kotlinlang.org/docs/coroutines-guide.html)

下一篇，将研究 Kotlin Flow。
