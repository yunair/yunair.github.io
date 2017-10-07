---
title: Retrofit2源码分析-(下)
date: 2016-05-22
tags: 
  - retrofit
  - square
---

### 引言
  上篇我们主要看了一下Retrofit的设计者对Retrofit1.+版本设计的评价（好的方面和坏的方面），同时也讲了Retrofit2都这些问题时如何解决的，这篇，我们就一起深入去了解一下。


### 网络请求
  一个网络连接需要构造请求，发送请求，处理服务端返回内容三个步骤组成，但是处理服务端返回内容是用库的人的工作，所以一个网络请求库只要做到前两步并将服务端返回内容转化为使用库的人需要的格式即可。

对于这三个步骤，两个版本的区别在于

#### 构造请求:

1. Retrofit2新加了`@Url`注解。
2. 新加了对某个请求单独加入Http请求头，`@Headers`(用于方法)/`@Header`(用于函数参数)。

#### 发送请求:

1. 将网络请求交给了OkHttp

#### 将服务端返回内容转化为你需要的格式:

1. Retrofit2将可以同时拿到返回内容的Header和Body，Retrofit1则不可以
2. 使用Call返回值类型将同步和异步请求统一
3. 可以添加多个Converter
4. 可以添加多个返回值类型的解析机制，在Retrofit2中称为`CallAdapterFactory`使用`addCallAdapterFactory()`


我们接下来跟着官方的例子来看一下这些修改时如何实现的。

### 源码分析

~~~ java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
    .build();

GitHubService service = retrofit.create(GitHubService.class);
Call<List<Repo>> repos = service.listRepos("octocat");

public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
~~~

从这里可以看出，通过`Retrofit`对象的`create()`方法创建出`GithubService`接口的实现，然后通过该实现进行网路请求，那么我们看看`create()`方法，

~~~ java
public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), 
                        new Class<?>[] {service},new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
}
~~~



熟悉Retrofit1的同学可以看出这里有3点不同的地方，不熟悉也没关系，因为相同的地方就是验证接口的有效性和使用动态代理生成接口对应的实现类。接下来讲一下这三点不同。

1. 添加了`validateEagerly`参数，让客户端调用网络请求的时候不需要在反射生成的代理类中才进行初始化;提前初始化，提前验证接口是否合法，不需要在调用时才知道。
2. 添加了一个暂时不用的方法，为以后使用做扩展`platform.isDefaultMethod(method)`。
3. 将初始化过的方法交给`OkHttpCall`去执行，执行完后，通过设置的`CallAdapter`将返回值转化为你要的返回类型对象，当你通过该对象调用它的方法的时候，将把请求交给`OkHttp`去执行。

分开来讲一下，先看第二点：

~~~ java
 boolean isDefaultMethod(Method method) {
    return false;
 }
~~~

可以看出这个方法压根就没有用，所以我不是很了解放在这里一个不需要的方法意义是什么。

再看第一点：

~~~ java
if (validateEagerly) {
   eagerlyValidateMethods(service);
}

private void eagerlyValidateMethods(Class<?> service) {
  //和Retrofit1一样，通过反射确定客户端的平台，使用不同的方式处理
  Platform platform = Platform.get();
  for (Method method : service.getDeclaredMethods()) {
    if (!platform.isDefaultMethod(method)) {
      loadServiceMethod(method);
    }
  }
}
//和Retrofit1一样，使用Map做了一个缓存，有的话直接取，
//否则遍历该方法的信息，创建ServiceMethod对象
ServiceMethod loadServiceMethod(Method method) {
    ServiceMethod result;
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
}
~~~

看一下第三点：

~~~ java
	ServiceMethod serviceMethod = loadServiceMethod(method);
    OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
    return serviceMethod.callAdapter.adapt(okHttpCall);
~~~

`serviceMethod.callAdapter.adapt()`方法将`Call`对象转化为你写的返回值类型对象，即你在接口中所需的除`Call<T>`类型外，都需要对应的`CallAdapter`来进行转化。`OkHttpCall`实现了`Call<T>`接口，也就是前一篇讲到的，加入了`Call<T>`返回值类型。默认的`CallAdapter`支持转化成`Call`类型对象，如果需要其他的类型，你就需要调用`addCallAdapterFactory()`方法来添加新类型了。然后通过`OkHttp`进行网络请求。

到这里大体上就讲完了，接下来就重新回到前面讲的，对一个网络请求的三个步骤进行分析，看看代码中具体是如何实现的（老规矩，忽视不重要的错误处理）。

#### 构造请求:

这一点的重点在`ServiceMethod`这个类，前面讲到了这个类初始化的地方，那么就从
`new ServiceMethod.Builder(this, method).build()`开始。
构造函数就初始化了一些类变量，来看一下`build()`方法:

~~~ java
public ServiceMethod build() {
	// 
    callAdapter = createCallAdapter();
    // 把函数返回值类型中泛型内的类型拿出来，
    // 比如接口方法中某个方法返回Call<Repo>，这个方法执行结果是Repo
    responseType = callAdapter.responseType();
    
    responseConverter = createResponseConverter();
    for (Annotation annotation : methodAnnotations) {
      //解析定义的Http注解，如GET，HTTP，FormUrlEncoded，Multipart，Headers
      //如果存在Headers注解，Headers注解内的内容不能为空
      parseMethodAnnotation(annotation);
    }
	// parameterAnnotationsArrays = method.getParameterAnnotations(),
	// 即参数中的注解，二维注解数组Annotation[][]
    int parameterCount = parameterAnnotationsArray.length;
    // ParameterHandler这个类如名字所示，负责处理有注解的函数参数
    parameterHandlers = new ParameterHandler<?>[parameterCount];
    for (int p = 0; p < parameterCount; p++) {
      // parameterType = method.getGenericParameterTypes()
      Type parameterType = parameterTypes[p];
      // 这个方法就是检测方法参数类型是否是可以处理的类型
      // 这个方法可以学到如何遍历所有的参数类型，包括泛型类型参数
	  if (Utils.hasUnresolvableType(parameterType)) {
	            throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
	                parameterType);
	          }

      Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
      // 这里处理Retrofit2定义的所有@Target(PARAMETER)的参数，返回对应的ParameterHandler
      parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
    }
    
    return new ServiceMethod<>(this);
}
~~~

除了`createCallAdapter()`，`createResponseConverter()`这两个方法，这两个方法我们一会再说，按名字理解就是创建`CallAdapter`和`ResponseConverter`，这两个东西的作用就是将方法的返回值转化成你要的。

剩下的关键方法都在方法中做了注释，这样，`ServiceMethod`类对象就生成了。
到了这里，先不继续进行，先看一下`parseMethodAnnotation(annotation)`方法中使用的`parseHttpMethodAndPath()`方法，这个方法中会做一些验证，这个验证个人觉得有些不好理解，所以放出源代码，并将注释写在方法中。

~~~ java
private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
  this.httpMethod = httpMethod;
  this.hasBody = hasBody;

  if (value.isEmpty()) {
    return;
  }

  // Get the relative URL path and existing query string, if present.
  int question = value.indexOf('?');
  if (question != -1 && question < value.length() - 1) {
    // Ensure the query string does not have any named parameters.
    String queryParams = value.substring(question + 1);
    Matcher queryParamMatcher = PARAM_URL_REGEX.matcher(queryParams);
    if (queryParamMatcher.find()) {// url中?之后的的String不允许存在"{}"
      throw methodError("URL query string \"%s\" must not have replace block. "
          + "For dynamic query parameters use @Query.", queryParams);
    }
  }

  this.relativeUrl = value;
  this.relativeUrlParamNames = parsePathParameters(value);
}
~~~

好了， 这点不好理解的地方也过去了。既然生成了`ServiceMethod`，那么我们继续往下看：
构造函数生成了`OkHttpCall`对象，最后，通过`ServiceMethod`的`CallAdapter`对象将`OkHttpCall`对象转化为你在网络请求接口中声明的返回值类型。

在这里，我们再看一下上面提到但没分析的那两个方法`createCallAdapter()`和`createResponseConverter()`。先看`createCallAdapter()`:

~~~ java
private CallAdapter<?> createCallAdapter() {
  // 返回方法声明的返回类型
  Type returnType = method.getGenericReturnType();
  
  if (returnType == void.class) {
    throw methodError("Service methods cannot return void.");
  }
  Annotation[] annotations = method.getAnnotations();
  return retrofit.callAdapter(returnType, annotations);
}
~~~

这个方法就获取了方法的返回值类型和方法的注解，最后调用`Retrofit`对象的`callAdapter()`方法找到对应的callAdapter，好了，我们继续,
retrofit对象的`callAdapter()`方法就一句代码，`return nextCallAdapter(null, returnType, annotations);`

~~~ java
public CallAdapter<?> nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
    Annotation[] annotations) {
  checkNotNull(returnType, "returnType == null");
  checkNotNull(annotations, "annotations == null");
  // List<CallAdapter.Factory> adapterFactories,你调用addCallAdapterFactory()就将你的CallAdapterFactory加入到了这个列表中
  //即使你不调用addCallAdapterFactory()方法，系统也会加入一个默认的CallAdapterFactory,就是在第一篇文章中提到的Call返回值类型。
  int start = adapterFactories.indexOf(skipPast) + 1;
  for (int i = start, count = adapterFactories.size(); i < count; i++) {
    CallAdapter<?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
    if (adapter != null) { // 如果返回值类型和adapter能处理的相同，则不为null
      return adapter;
    }
  }
  throw new IllegalArgumentException("...");
}
~~~

这里将注解传给`callAdapter()`是为了我们使用更好的扩展。
这个扩展具体指什么呢？前面我们提到，可以添加多个`Converter`和`CallAdapter`。
这里就是给你这样一种能力，如果你有自定义注解，在这里你就可以拿到自己定义的注解来处理。
`callAdapter()`方法直接从`adapterFactories`列表中获取目标类型，即直接获取你需要该方法返回什么类型的值。
可以看到，这里就是找哪个`CallAdapter`可以处理用户声明的该方法返回值类型，这样一个`CallAdapter`对象就生成了。

接下来看看`createResponseConverter()`方法，在第一篇的介绍中提到`Converter`和`CallAdapter`类似，那么我猜测代码中的创建过程应该也是类似的，来仔细瞧一瞧是不是这样。

~~~ java
private Converter<ResponseBody, T> createResponseConverter() {
  Annotation[] annotations = method.getAnnotations();
  try {
    return retrofit.responseBodyConverter(responseType, annotations);
  } catch (RuntimeException e) { // Wide exception range because factories are user code.
    throw methodError(e, "Unable to create converter for %s", responseType);
  }
}
~~~

和上面一样，调用了retrofit对象的方法，`responseBodyConverter()`方法同样一句代码:
`return nextResponseBodyConverter(null, type, annotations);`

~~~ java
public <T> Converter<ResponseBody, T> nextResponseBodyConverter(Converter.Factory skipPast,
    Type type, Annotation[] annotations) {
  checkNotNull(type, "type == null");
  checkNotNull(annotations, "annotations == null");
  // List<Converter.Factory> converterFactories,你调用addConverterFactory()方法就是将你的Converter.Factory加入这个列表中，因为列表有序，所以从前往后找，JSON没有明显的特征，所以需要将JSON放在最后
  // 即使你不调用addConverterFactory()方法，会加入内置的Converter.Factory
  // 下面是加入内置Converter.Factory的方法和注释。
   // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      // converterFactories.add(new BuiltInConverters());
  int start = converterFactories.indexOf(skipPast) + 1;
  for(int i = start, count = converterFactories.size(); i < count; i++){
    //将ResponseBody转化为你需要的类型
    Converter<ResponseBody, ?> converter =
        converterFactories.get(i).responseBodyConverter(type, annotations, this);
    if (converter != null) { // 如果能处理对应的type，则不为null
      //noinspection unchecked
      return (Converter<ResponseBody, T>) converter;
    }
  }
  throw new IllegalArgumentException("...");
}
~~~

除了这个ResponseBodyConverter，还有一个RequestBodyConverter，其实这个也是从`converterFactories`里面寻找处理请求参数的Converter。

到这里，`Retrofit`对象的`create()`方法就分析完了，`CallAdapter`将`OkHttpCall`对象转化成了你想要的对象，此时就可以拿你想要的对象进行操作了，也就是构造请求完成了。

#### 发送请求:

当用户调用接口中方法的时候，底层直接通过OkHttp进行发送请求。

#### 将服务端返回内容转化为你需要的格式:

OkHttp接收到请求后，通过`Converter`将服务端返回内容转化为你需要的格式。所以到这里源码的流程已经走完了。

可以看出其实这个框架在构造请求的完成的时候工作也完成了。

接下来有一些需要注意的地方:

1. 接口中任何一个方法的只允许被一个Http方法注解(即[GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS, HTTP]这个列表中的元素每个方法只能使用一个)
2. 网络请求只要服务端有数据返回，就不算failure，Retrotfit1的时候404和500都算failure。所以当你使用Callback时，在`onResponse`函数里，使用`response.isSuccessful()`确定服务端返回正常数据还是服务端错误信息。
3. Url的拼接规则变了，初始化Retrofit对象的时候baseUrl最后一定要加'/'。详细的内容查看下面的表格:



|           BaseUrl            |  AnnotationUrl  |               RequestUrl               |
| :--------------------------: | :-------------: | :------------------------------------: |
| https://api.github.com/repo/ |   /square/end   |   https://api.github.com/square/end    |
| https://api.github.com/repo/ |   square/end    | https://api.github.com/repo/square/end |
| https://api.github.com/repo/ | http://z.cn/end |            http://z.cn/end             |



最后放一张流程图，来源[Retrofit分析-漂亮的解耦套路][2]，是一个很棒的流程图，可以加深对此源码分析的理解。

![](/img/retrofit/retrofit2.png)

最后是一些参考资料:

1. [你真的会用Retrofit2吗?][1] 对Retrofit2中用到的注解进行了分类，可以让你更清晰的使用对应的注解；提供了自定义`ConverterFactory`和`CallAdapterFactory`的思路。

[1]: http://www.jianshu.com/p/308f3c54abdd
[2​]: http://www.jianshu.com/p/45cb536be2f4#

