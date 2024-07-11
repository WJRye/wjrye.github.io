---
layout: post
title: Retrofit 中的注解以及如何自定义接口方法注解
categories: [Android]
description: 在 Retrofit 中自定义接口方法注解主要是利用了CallAdapter.Factory，并在它的get方法中拿到方法注解进行解析。注意，这里并不能自定义接口方法参数注解。
keywords: Android, Retrofit
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 接口方法和方法参数注解

[Retrofit](https://github.com/square/retrofit) 框架中一个最大的特点就是**用注解来描述 HTTP 请求**：

- 支持 URL 参数替换和查询参数
- 将对象转换为请求主体（例如，JSON、协议缓冲区）
- 多部分请求正文和文件上传

注解分为方法注解和方法参数注解，它标志着一个请求将会被如何处理。

### 1.方法注解

#### 1.1 注解描述请求方式

| 注解名字     | 表示意思      |
| -------- | --------- |
| @GET     | GET请求     |
| @POST    | POST请求    |
| @DELETE  | DELETE请求  |
| @HEAD    | HEAD请求    |
| @OPTIONS | OPTIONS请求 |
| @PATCH   | PATCH请求   |

##### 例子

1.注解`@GET`

```
    @GET("/users/query/all")
    Call<List<User>> getUsers();
```

2.注解`@POST`：

```
    @POST("/users/query")
    @FormUrlEncoded
    Call<User> getUser(@Field("id") String id);
```

#### 1.2 注解描述请求内容类型、返回结果和请求头

| 注解名字            | 表示意思                                                                                                                                                                                                   |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| @FormUrlEncoded | 该注解与方法注解 @POST 和参数注解 @Field 一起使用，标志着请求实体（body部分）内容会经过 URL 编码后上传，以及该请求（POST）下的 Content-Type 为 " application/x-www-form-urlencoded " 。换一句话说，就是请求的参数以 key1=value1&key2=value2.. 的方式发送到服务端，并且参数会经过 URL 编码。 |
| @Multipart      | 该注解与方法注解 @POST 和参数注解 @Part 一起使用，标志着请求实体（body部分）内容使用表单上传，以及该请求（POST）下的 Content-Type 为 " multipart/form-data " 。使用表单上传的数据可以是纯文本，纯文件，也可以是文本+文件。                                                           |
| @Streaming      | 将返回结果（ResponseBody）转换为流（字节）。                                                                                                                                                                           |
| @Headers        | 标志请求头。                                                                                                                                                                                                 |

##### 例子

1.注解`@FormUrlEncoded`

发送`POST`请求，请求头`Content-Type：application/x-www-form-urlencoded`，请求实体`id=value`。

```
    @POST("/users/query")
    @FormUrlEncoded
    Call<User> getUser(@Field("id") String id);
```

2.注解`@Multipart`

发送`POST`请求，请求头`Content-Type： multipart/form-data `，请求实体`id=value`和图片文件。

```
    @POST("/users/update")
    @Multipart
    Call<ResponseBody> updateUserPortrait(@Part("id") int id, @Part(value = "image", encoding = "8-bit") RequestBody image);
```

3.注解` @Streaming`

```
    @GET("/users/download")
    @Streaming
    Call<ResponseBody> downloadUserFile(@Query("id") int id);
```

4.注解`@Headers`

```
    @Headers("Cache-Control: max-age=640000")
    @GET("/users/download")
    @Streaming
    Call<ResponseBody> downloadUserFile(@Query("id") int id);
```

### 2.参数注解

| 注解名字       | 表示意思                                                                                                                         |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------- |
| @Body      | 该注解与方法注解 @POST 或 @PUT 一起使用，标志参数是一个POJO 或 RequestBody对象，最终会直接将参数对象转换为请求实体（body部分）内容。                                          |
| @Field     | 该注解与方法注解 @POST 和 @FormUrlEncoded 一起使用，标志着参数是一个键值对，也就是请求实体（body部分）是以key1=value1&key2=value2.. 或 key=value1&key=value2.. 方式传递。 |
| @FieldMap  | 与@Field注解一样，只不过参数对象是一个Map。                                                                                                   |
| @Header    | 标志参数是一个请求头                                                                                                                   |
| @HeaderMap | 标志参数是一个Map集合的请求头                                                                                                             |
| @Part      | 该注解与方法注解 @POST 和参数注解 @Multipart 一起使用                                                                                         |
| @PartMap   | 与@Part注解一样，只不过参数对象是一个Map，Map接受的类型是<String,RequestBody>                                                                       |
| @Path      | 标志参数是用来替换请求路径的                                                                                                               |
| @Query     | 标志参数（@Query("key1") String value1）会追加在请求路径后面（?key1=value1）。                                                                  |
| @QueryMap  | 与@Query注解一样，只不过参数对象是一个Map集合，会Map集合转换为键值对。                                                                                    |
| @QueryName | 与@Query注解相似，只不过参数（@QueryName String filter）会直接追加在路径后面（?filter）                                                               |
| @Tag       | 标志参数用来给这次请求打个Tag                                                                                                             |
| @Url       | 标志参数用来替换请求路径，会忽略baseUrl                                                                                                      |

#### 例子

1.注解`@Body`：

```
    @POST("/users/upload")
    Call<ResponseBody> uploadUserLog(@Body RequestBody body);

    //创建RequestBody
    RequestBody body = RequestBody.create(MediaType.parse("Content-Type: application/json;charset=UTF-8"), "Your json log");
    UserService#uploadUserLog(body);
```

2.注解`@Filed`和`@FiledMap`

注解`@Filed`：

```
   @POST("/users/query")
    @FormUrlEncoded
    Call<User> getUser(@Field("id") String id);
```

注解`@FieldMap`：

```
    @POST("/users/query")
    @FormUrlEncoded
    Call<User> getUser(@FieldMap Map<String, String> filedMap);


    Map<String,String> filedMap = new HashMap<>();
    filedMap.put("id", "123");
    filedMap.put("age", "25");
    UserService#getUser(filedMap);
```

2.注解`@Header`和`@HeaderMap`

注解`@Header`：

```
    @POST("/users/query")
    @FormUrlEncoded
    Call<User> getUser(@Field("id") String id, @Header("Accept-Language") String lang);
```

注解`@HeaderMap`：

```
    @POST("/users/query")
    @FormUrlEncoded
    Call<User> getUser(@Field("id") String id, @HeaderMap Map<String, String> headerMap);


    Map<String,String> headerMap = new HashMap<>();
    headerMap.put("Accept-Charset", "utf-8");
    headerMap.put("Accept", "text/plain");
    UserService#getUser("123", headerMap);
```

2.注解`@Part`和`@PartMap`

注解`@Part`：

```
    @POST("/users/update")
    @Multipart
    Call<ResponseBody> updateUserPortrait(@Part("id") int id, @Part(value = "image", encoding = "8-bit") RequestBody image);
```

注解`@PartMap`：

```
    @POST("/users/upload/feedback")
    @Multipart
    Call<ResponseBody> uploadUserFeedback( @Part("file") RequestBody file, @PartMap Map<String, RequestBody> params);


    Map<String, RequestBody> partMap = new HashMap<>();
    partMap.put("deviceInfo", RequestBody.create(MediaType.parse("text/plain"), "deviceInfo"));
    partMap.put("content", RequestBody.create(MediaType.parse("text/plain"), "content"));
    File file = new File("test.png");
    RequestBody requestBody = RequestBody.create(MultipartBody.FORM, file);
    UserService#uploadUserFeedback(requestBody, partMap);
```

3.注解`@Path`：

```
    @GET("/users/{name}")
    Call<List<User>> getUserByName(@Path("name") String name);
```

4.注解`@Query`和`@QueryMap`

注解`@Query`：

```
    @GET("/users/query/friends")
    Call<List<User>> getUserFriends(@Query("page") int page);
```

注解`@QueryMap`：

```
    @GET("/users/query/friends/filter")
    Call<List<User>> getUserFriends(@QueryMap Map<String, String> params);

     Map<String, String> queryMap = new HashMap<>();
     queryMap.put("age", "23");
     queryMap.put("sex", "male");
     UserService#.getUserFriends(queryMap);
```

5.注解`@QueryName`：

```
    @GET("/users/query/friends/filter")
    Call<List<User>> getUserFriendsByFileter(@QueryName String filter);
```

6.注解`@Url`：

```
    @GET("/users/config")
    Call<ResponseBody> getUserConfig(@Url String url);
```

----

## 注解解析过程

注解的解析是在类`ServiceMethod`中的静态内部类`Builder`中的`build`方法中进行的，通过调用`parseMethodAnnotation(annotation)`方法解析方法注解，调用`parseParameter`方法解析参数注解。

关于具体解析步骤，我们先使用`Retrofit`创建一个服务接口：

```
        Retrofit retrofit = new Retrofit.Builder().baseUrl("https://api.example.com/").build();
        UserService service = retrofit.create(UserService.class);
```

服务接口的创建使用了 **Java 动态代理**，具体看`Retrofit`类中的`create`方法：

```
  public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```

根据上一篇[Java 动态代理 Proxy.newProxyInstance()]({{site.url}}/2019/06/30/Java-Proxy-Intro/)，接口`InvocationHandler`中的回调方法`invoke`是在调用目标接口（UserService）的方法时候调用的，`invoke`方法的返回值就是目标接口（UserService）方法的返回值。如上面的例子注解`@GET`：

```
    @GET("/users/query/all")
    Call<List<User>> getUsers();

    Call<List<User>> call =  service.getUsers();
    call.enqueue(callback);
```

在 UserService 调用`getUsers()`方法的时候，就会执行`invoke`方法，并返回`Call<List<User>>`。

在`invoke`方法中，首先判断执行的方法是不是在类`Object`中声明的方法：

```
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
```

像`hashCode`、`equals`等都是`Object`里面的方法。其次是判断执行的方法是不是目标接口（UserService）中声明的默认方法（Java 8.0版本以上，可以在接口中声明默认方法）：

```
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
```

`Platform` 主要区分是Android 还是 Java。最后创建`ServiceMethod`、`OkHttpCall`对象，并通过`CallAdapter`对象中的`adapt`方法返回相应的接口方法返回值：

```
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
```

在`loadServiceMethod` 方法中，先查看该`method`对应的`ServiceMethod`是否已经存在，如果存在就直接返回，如果没有就创建一个并放到缓存中：

```
  ServiceMethod<?, ?> loadServiceMethod(Method method) {
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder<>(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

类`ServiceMethod`中的静态内部类`Builder`的构造方法接受`Retrofit`和`Method`对象，最后调用`build()`方法创建`ServiceMethod`对象：

```
    Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      this.methodAnnotations = method.getAnnotations();
      this.parameterTypes = method.getGenericParameterTypes();
      this.parameterAnnotationsArray = method.getParameterAnnotations();
    }

    public ServiceMethod build() {
      callAdapter = createCallAdapter();
      responseType = callAdapter.responseType();

      ...省略部分代码

      responseConverter = createResponseConverter();

      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation); //解析方法注解
      }

      ...省略部分代码

      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];
        if (Utils.hasUnresolvableType(parameterType)) {
          throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
              parameterType);
        }

        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        if (parameterAnnotations == null) {
          throw parameterError(p, "No Retrofit annotation found.");
        }

        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations); //解析方法参数注解
      }

      ...省略部分代码

      return new ServiceMethod<>(this);
    }
```

`parseMethodAnnotation(annotation)` 方法里面主要解析方法注解：

```
    private void parseMethodAnnotation(Annotation annotation) {
      if (annotation instanceof DELETE) {
        parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
      } else if (annotation instanceof GET) {
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
      } else if (annotation instanceof HEAD) {
        parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
        if (!Void.class.equals(responseType)) {
          throw methodError("HEAD method must use Void as response type.");
        }
      } else if (annotation instanceof PATCH) {
        parseHttpMethodAndPath("PATCH", ((PATCH) annotation).value(), true);
      } else if (annotation instanceof POST) {
        parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
      } else if (annotation instanceof PUT) {
        parseHttpMethodAndPath("PUT", ((PUT) annotation).value(), true);
      } else if (annotation instanceof OPTIONS) {
        parseHttpMethodAndPath("OPTIONS", ((OPTIONS) annotation).value(), false);
      } else if (annotation instanceof HTTP) {
        HTTP http = (HTTP) annotation;
        parseHttpMethodAndPath(http.method(), http.path(), http.hasBody());
      } else if (annotation instanceof retrofit2.http.Headers) {
        String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
        if (headersToParse.length == 0) {
          throw methodError("@Headers annotation is empty.");
        }
        headers = parseHeaders(headersToParse);
      } else if (annotation instanceof Multipart) {
        if (isFormEncoded) {
          throw methodError("Only one encoding annotation is allowed.");
        }
        isMultipart = true;
      } else if (annotation instanceof FormUrlEncoded) {
        if (isMultipart) {
          throw methodError("Only one encoding annotation is allowed.");
        }
        isFormEncoded = true;
      }
    }
```

`parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations)` 方法里面主要解析方法参数注解：

```
    private ParameterHandler<?> parseParameter(
        int p, Type parameterType, Annotation[] annotations) {
      ParameterHandler<?> result = null;
      for (Annotation annotation : annotations) {
        ParameterHandler<?> annotationAction = parseParameterAnnotation(
            p, parameterType, annotations, annotation);

        if (annotationAction == null) {
          continue;
        }

        if (result != null) {
          throw parameterError(p, "Multiple Retrofit annotations found, only one allowed.");
        }

        result = annotationAction;
      }

      if (result == null) {
        throw parameterError(p, "No Retrofit annotation found.");
      }

      return result;
    }

    private ParameterHandler<?> parseParameterAnnotation(
        int p, Type type, Annotation[] annotations, Annotation annotation) {
      if (annotation instanceof Url) {
        ...省略部分代码

      } else if (annotation instanceof Path) {
        ...省略部分代码

      } else if (annotation instanceof QueryName) {
        ...省略部分代码

      } else if (annotation instanceof QueryMap) {
        ...省略部分代码

      } else if (annotation instanceof Header) {
        ...省略部分代码

      } else if (annotation instanceof HeaderMap) {
        ...省略部分代码

      } else if (annotation instanceof Field) {
        ...省略部分代码

      } else if (annotation instanceof FieldMap) {
        ...省略部分代码

      } else if (annotation instanceof Part) {
        ...省略部分代码

      } else if (annotation instanceof PartMap) {
        ...省略部分代码

      } else if (annotation instanceof Body) {
        ...省略部分代码

      }
    }
```

注解具体解析细节这里不再分析介绍，有兴趣可以仔细阅读这部分代码。

----

## 如何自定义接口方法注解

自定义接口方法注解有一个关键的问题就是：如何去解析以及在什么时候解析这个方法注解？

从上面注解解析过程，知道注解的解析是在类`ServiceMethod`中的静态内部类`Builder`中的`build`方法中进行的。再看在`build`方法中的第一行代码：

```
    callAdapter = createCallAdapter(); //创建一个 CallAdapter 对象
```

`createCallAdapter()` 方法创建的是一个 `CallAdapter<T, R>` 对象。 `CallAdapter<R, T>`  是一个接口，它的作用是将请求结果响应类型`R`转换为自定义的类型`T`。`CallAdapter` 对象 是通过`CallAdapter.Factory`类来创建的。它是在创建`Retrofit`对象的时候，通过`Retrofit.Builder`的
`#addCallAdapterFactory(Factory)`方法将它添加进来的：

```
        Retrofit retrofit = Retrofit.Builder()
                .baseUrl("https://api.example.com/")
                .client(okHttpClient)
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .addConverterFactory(GsonConverterFactory.create(createGsonConverter()))
                .build();
        UserService service = retrofit.create(UserService.class);
```

再看`createCallAdapter`方法：

```
  private CallAdapter<T, R> createCallAdapter() {
      Type returnType = method.getGenericReturnType();//得到方法返回值类型
      if (Utils.hasUnresolvableType(returnType)) {
        throw methodError(
            "Method return type must not include a type variable or wildcard: %s", returnType);
      }
      if (returnType == void.class) {
        throw methodError("Service methods cannot return void.");
      }
      Annotation[] annotations = method.getAnnotations();//获得方法注解
      try {
        //noinspection unchecked
        return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
      } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create call adapter for %s", returnType);
      }
    }
```

在方法的最后，通过`Retrofit`的`callAdapter`方法返回一个`CallAdapter<T, R>`对象，这个`callAdapter`方法接受两个参数，它们分别是方法返回值类型` Type returnType = method.getGenericReturnType();` 和 方法注解` Annotation[] annotations = method.getAnnotations();`。**如果要自定义注解，就要想办法拿到`annotations`这个参数。**

再来看`callAdapter`方法：

```
  final List<CallAdapter.Factory> adapterFactories;

  public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
  }

  public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {

    ...省略部分代码

    int start = adapterFactories.indexOf(skipPast) + 1;//默认会添加一个`CallAdapter<?, ?>`对象
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      CallAdapter<?, ?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }

    ...省略部分代码
  }
```

在`nextCallAdapter`方法中，通过遍历`adapterFactories`中的`CallAdapter.Factory`的 `get` 方法得到`CallAdapter<?, ?>`对象：

```
 public abstract @Nullable CallAdapter<?, ?> get(Type returnType, Annotation[] annotations,
        Retrofit retrofit);
```

`get` 方法中有三个参数`Type`、`Annotation[]`和`Retrofit`对象，**其中`Annotation[]`对像就是接口方法的方法注解，如果有自定义方法注解，那么这个自定义方法注解就在里面。所以自定义注解就是要实现``CallAdapter.Factory``接口，然后通过`Retrofit.Builder`的
`#addCallAdapterFactory(Factory)`方法将它添加进来。**

下面看下`Retrofit`框架提供的默认`CallAdapter.Factory，它是在在创建`Retrofit` 对象的时候，通过`Retrofit.Builder#build` 方法添加的`：

```
      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));
```

再看下 `platform` 的 `defaultCallAdapterFactory`方法：

```
  CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
    if (callbackExecutor != null) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }
    return DefaultCallAdapterFactory.INSTANCE;
  }
```

 如果有设置`Retrofit.Builder#build.callbackExecutor(Executor executor)`，就使用`ExecutorCallAdapterFactory`，否则就使用`DefaultCallAdapterFactory`。

下面看看类`DefaultCallAdapterFactory`：

```
final class DefaultCallAdapterFactory extends CallAdapter.Factory {
  static final CallAdapter.Factory INSTANCE = new DefaultCallAdapterFactory();

  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }

    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public Call<Object> adapt(Call<Object> call) {
        return call;
      }
    };
  }
}
```

根据上面代码，就知道如何创建一个`CallAdapter.Factory`，并在它的`get`方法中解析自定义的接口方法注解。

### 例子

自定义一个标识这个请求接口是不是需要加密的注解：

```
/**
 * 加密注解，在接口方法中添加了这个注解，就代表着这个请求接口传输的数据会进行加密
 */
public @interface Encryption {

}
```

自定义一个`CallAdapter.Factory`：

```
public class EncryptionCallAdapterFactory extends CallAdapter.Factory {
    private static final CallAdapter.Factory INSTANCE = new EncryptionCallAdapterFactory();

    public static CallAdapter.Factory create() {
        return INSTANCE;
    }

    @Override
    public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
        if (getRawType(returnType) != Call.class) {
            return null;
        }
        final Type responseType = Utils.getCallResponseType(returnType);

        String baseUrl = retrofit.baseUrl().url().toString();
        String path = null;//请求路径
        boolean hasEncryption = false;//是否有加密注解Encryption
        for (Annotation annotation : annotations) {
            if (annotation instanceof GET) {
                path = ((GET) annotation).value();
            } else if (annotation instanceof POST) {
                path = ((POST) annotation).value();
            } else if (annotation instanceof Encryption) {
                hasEncryption = true;
            }
        }
        if (path != null && hasEncryption) {
            //do something
        }

        return new CallAdapter<Object, Call<?>>() {
            @Override
            public Type responseType() {
                return responseType;
            }

            @Override
            public Call<Object> adapt(Call<Object> call) {
                return call;
            }
        };
    }
}
```

在`get`方法中，拿到自定义的接口方法注解，然后进行解析。上面对请求接口加密，在`get`方法中只能做到标识下这个请求接口的请求实体需要加密，接下来还需要在`OKHttpClient` 的 拦截器`Interceptor`中对请求实体进行加密处理。

添加`EncryptionCallAdapterFactory`：

```
        Retrofit retrofit = Retrofit.Builder()
                .baseUrl("https://api.example.com/")
                .client(okHttpClient)
                .addCallAdapterFactory(EncryptionCallAdapterFactory.create())
                .addConverterFactory(GsonConverterFactory.create(createGsonConverter()))
                .build();
        UserService service = retrofit.create(UserService.class);
```

**总结，自定义接口方法注解主要是利用了`CallAdapter.Factory`，并在它的`get`方法中拿到方法注解进行解析。注意，这里并不能自定义接口方法参数注解。**
