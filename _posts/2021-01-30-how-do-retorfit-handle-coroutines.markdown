---
layout:     post
title:      "Retrofit 如何处理协程"
subtitle:   "简洁的代码不简单的思维方法"
date:       2021-01-30
author:     "ChengTao"
header-img: "img/android.png"
tags:
    - Android
---

> Retofit 在 2.6.0 版本增加了对 Koltin suspend 方法的支持，这对于使用 Kotlin
开发的小伙伴简直不要太友好了，我们今天就带着疑问看看 Retrofit
是如何实现这一功能的。

## 快速上手

```kotlin
@GET("users/{id}")
suspend fun user(@Path("id") id: Long): User
```

可以看到 Retrofit 这波对协程的支持简直太简洁了，只需要给方法添加 suspend 关键字即可，
以后又可以少很多行代码了，哈哈哈。

## 初入 suspend 方法

在了解 Retrofit 如何实现对协程的支持之前，我们必须先了解下 supend 方法，这样才能
帮助我们更好地理解源码，我们先看看下面的这个这个例子。

```kotlin
// 非 suspend 方法
fun test(){}

// suspend 方法
suspend fun testSuspend(){}

// 带参数的 suspend 方法
suspend fun testSuspend(text : String){}
```

其实在代码层面我们是看不出 suspend 方法和非 suspend 方法有多么大的区别的，除了有
一个 supsend 关键字，其他的毫无差别，这个时候我们就需要看看这几个方法字节码了。

```text
Compiled from "SuspendTest.kt"
public final class SuspendTestKt {
  public static final void test();
    Code:
       0: return

  public static final java.lang.Object testSuspend(kotlin.coroutines.Continuation<? super kotlin.Unit>);
    Code:
       0: getstatic     #17                 // Field kotlin/Unit.INSTANCE:Lkotlin/Unit;
       3: areturn

  public static final java.lang.Object testSuspend(java.lang.String, kotlin.coroutines.Continuation<? super kotlin.Unit>);
    Code:
       0: getstatic     #17                 // Field kotlin/Unit.INSTANCE:Lkotlin/Unit;
       3: areturn
}
```

从上面反汇编的结果我们可以清晰的看出，suspend 方法比非 suspend 方法多一个参数，
并且这个参数在参数列表的最后一个，参数类型是 **Continuation**, 了解到这一区别我
们就可以着手看一下 Retrofit 的源码了。

## 如何判断方法是 suspend 方法

通过解析接口方法，Retrofit 创建了一个 RequestFactory 用于之后创建 OKhttp 的
Request, 而这个 RequestFactory 类多了一个 `isKotlinSuspendFunction` 的属性，
用于标记该方法是否为 suspend 方法，具体解析过程看下面的源码。

```java
RequestFactory build() {
  ...

  int parameterCount = parameterAnnotationsArray.length;
  parameterHandlers = new ParameterHandler<?>[parameterCount];
  for (int p = 0, lastParameter = parameterCount - 1; p < parameterCount; p++) {
	// p == lastParameter 是标记是否对最后一个参数进行 Continuation 类型地检测
	// 因为我们知道 supsend 方法的最后一个参数是 Continuation 类型
	parameterHandlers[p] =
		parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p], p == lastParameter);
  }

  ...

  return new RequestFactory(this);
}

private @Nullable ParameterHandler<?> parseParameter(
	int p, Type parameterType, @Nullable Annotation[] annotations, boolean allowContinuation) {

  ...

  if (result == null) {
	if (allowContinuation) {
	  try {
		// 判断参数的类型是否为 Continuation 类型
		if (Utils.getRawType(parameterType) == Continuation.class) {
		  // 标记当前的方法是 supend 方法
		  isKotlinSuspendFunction = true;
		  return null;
		}
	  } catch (NoClassDefFoundError ignored) {
	  }
	}
	throw parameterError(method, p, "No Retrofit annotation found.");
  }

  return result;
}
```

截止目前我们已经知道 Retrofit 是如何判断当前的方法为 suspend 方法了，简单概括下，
就是判断方法最后一个参数的类型是不是 Continuation 类型.

## 如何判断方法的返回值

除了需要判断方法是否为 suspend 方法之外，还需要判断方法的返回类型是什么，要不然
Retrofit 就无法创建相应的 CallAdapter.

在此之前我们还需要反汇编一个带返回值的 suspend 方法：

```kotlin
suspend fun testString() : String {
	return ""
}
```

```text
Compiled from "SuspendTest.kt"
public final class SuspendTestKt {
  public static final java.lang.Object testString(kotlin.coroutines.Continuation<? super java.lang.String>);
    Code:
       0: ldc           #27                 // String
       2: areturn
}
```

从上面的反汇编结果不难看出，suspend 方法的返回值其实就是 Continuation 的泛型的
具体类型，所以只要获取 Continuation 的泛型的具体类型，我们就可以知道 suspend 方法
的返回类型，Retrofit 也是这么做的，我们具体看源码。

```java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
  Retrofit retrofit, Method method, RequestFactory requestFactory) {
	boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;
	boolean continuationWantsResponse = false;
	boolean continuationBodyNullable = false;

	Annotation[] annotations = method.getAnnotations();
	Type adapterType;
	// 判断该方法是否为 kotlin 的 suspend 方法
	if (isKotlinSuspendFunction) {
	  Type[] parameterTypes = method.getGenericParameterTypes();
	  // 获取最后一个参数的第一个泛型类型
	  // 其实 Continuation 就一个泛型，这里 responseType 就是 suspend 方法的返回类型
	  Type responseType = Utils.getParameterLowerBound(0,
		  (ParameterizedType) parameterTypes[parameterTypes.length - 1]);
	  // 再判断返回类型是否为 Response, 这里这么判断是为了方便开发者，我们看下下面的列子：
	  //
	  // suspend getUser() : User 
	  // suspend getUserResponse : Response<User>
	  // 
	  // 如果不是 Response 的话，Retrofit 就将直接返回我们的数据类型，因为 Response
	  // 的判断大同小异，就是判断请求是否成功，如果不成功就抛异常，而 Retrofit 直接
	  // 帮我们省去了这一步，可以说相当 Nice 了。
	  if (getRawType(responseType) == Response.class && responseType instanceof ParameterizedType) {
		// Unwrap the actual body type from Response<T>.
		responseType = Utils.getParameterUpperBound(0, (ParameterizedType) responseType);
		continuationWantsResponse = true;
	  } else {
		// TODO figure out if type is nullable or not
		// Metadata metadata = method.getDeclaringClass().getAnnotation(Metadata.class)
		// Find the entry for method
		// Determine if return type is nullable or not
	  }
	  
	  // 为 suspend 方法创建 CallAdapter 的类型，我们知道 Retrofit 默认情况只支持
	  // Call 类型，比如下面的例子：
	  //
	  // Call<User> getUser();
	  //
	  // Retrofit 非常聪明地将 suspend 方法的 CallAdapter 类型转化成了 Call 类型，
	  // 这里 ParameterizedTypeImpl 就是一个自定的 Call 类型。
	  adapterType = new Utils.ParameterizedTypeImpl(null, Call.class, responseType);
	  annotations = SkipCallbackExecutorImpl.ensurePresent(annotations);
	} else {
	  adapterType = method.getGenericReturnType();
	}

	// 通过 adapterType 创建该方法的 CallAdapter, 这里 supsend 方法的 adapterType 为
	// Call 类型
	CallAdapter<ResponseT, ReturnT> callAdapter =
		createCallAdapter(retrofit, method, adapterType, annotations);
	Type responseType = callAdapter.responseType();
	if (responseType == okhttp3.Response.class) {
	  throw methodError(method, "'"
		  + getRawType(responseType).getName()
		  + "' is not a valid response body type. Did you mean ResponseBody?");
	}
	if (responseType == Response.class) {
	  throw methodError(method, "Response must include generic type (e.g., Response<String>)");
	}
	// TODO support Unit for Kotlin?
	if (requestFactory.httpMethod.equals("HEAD") && !Void.class.equals(responseType)) {
	  throw methodError(method, "HEAD method must use Void as response type.");
	}

	Converter<ResponseBody, ResponseT> responseConverter =
		createResponseConverter(retrofit, method, responseType);

	okhttp3.Call.Factory callFactory = retrofit.callFactory;
	if (!isKotlinSuspendFunction) {
	  return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
	} else if (continuationWantsResponse) {
	  //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
	  // 如果 suspend 的返回类型为 Response 返回 SuspendForResponse
	  return (HttpServiceMethod<ResponseT, ReturnT>) new SuspendForResponse<>(requestFactory,
		  callFactory, responseConverter, (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter);
	} else {
	  //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
	  // 如果 suspend 的返回类型为具体数据类型返回 SuspendForBody
	  return (HttpServiceMethod<ResponseT, ReturnT>) new SuspendForBody<>(requestFactory,
		  callFactory, responseConverter, (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter,
		  continuationBodyNullable);
	}
}
```

## 如何处理 suspend 方法的返回值

从上面的介绍我们知道，Retrofit 支持两种类型返回值的 suspend 方法，一种是返回值
类型为 Response 类型，另一种是具体数据类型，那针对这两种类型 Retrofit 又是如何
处理的呢？对于 Response 类型返回值的方法 Retrofit 返回了一个 SuspendForResponse
的 HttpServiceMethod, 而对于具体数据类型返回值的方法 Retrofit 返回了一个
SuspendForBody 的 HttpServiceMethod, 具体细节我们接着看源码：

### 处理返回值为 Response 的 suspend 方法

```java
static final class SuspendForResponse<ResponseT> extends HttpServiceMethod<ResponseT, Object> {
	private final CallAdapter<ResponseT, Call<ResponseT>> callAdapter;

	SuspendForResponse(RequestFactory requestFactory, okhttp3.Call.Factory callFactory,
		Converter<ResponseBody, ResponseT> responseConverter,
		CallAdapter<ResponseT, Call<ResponseT>> callAdapter) {
	  super(requestFactory, callFactory, responseConverter);
	  this.callAdapter = callAdapter;
	}

	@Override protected Object adapt(Call<ResponseT> call, Object[] args) {
	  // 参数里的 call 为 OkHttpCall
	  call = callAdapter.adapt(call);

	  //noinspection unchecked Checked by reflection inside RequestFactory.
	  // 获取最后一个参数 Continuation, 用于接收返回结果
	  Continuation<Response<ResponseT>> continuation =
		  (Continuation<Response<ResponseT>>) args[args.length - 1];
	  // 等待 Response 的返回
	  return KotlinExtensions.awaitResponse(call, continuation);
	}
}
```

```kotlin
suspend fun <T> Call<T>.awaitResponse(): Response<T> {
  return suspendCancellableCoroutine { continuation ->
    continuation.invokeOnCancellation {
      cancel()
    }
    // 执行 Call 的 enque 方法，里面具体的操作就是通过之前的 RequestFactory 创建
    // 的 OkHttp 的 Request, 再通过 Request 创建 OkHttp 的 Call, 最后执行 OkHttp 的
    // Call 的 enque 方法。使用了代理模式，具体的请求是交给 OkHttp 完成的。
    enqueue(object : Callback<T> {
      override fun onResponse(call: Call<T>, response: Response<T>) {
        // 成功返回结果
        continuation.resume(response)
      }

      override fun onFailure(call: Call<T>, t: Throwable) {
        // 失败返回异常
        // 所以执行 suspend 方法的时候我们需要 try catch 去捕获异常。
        continuation.resumeWithException(t)
      }
    })
  }
}
```

### 处理返回值为具体数据类型的 suspend 方法

```java
static final class SuspendForBody<ResponseT> extends HttpServiceMethod<ResponseT, Object> {
	private final CallAdapter<ResponseT, Call<ResponseT>> callAdapter;
	private final boolean isNullable;

	SuspendForBody(RequestFactory requestFactory, okhttp3.Call.Factory callFactory,
		Converter<ResponseBody, ResponseT> responseConverter,
		CallAdapter<ResponseT, Call<ResponseT>> callAdapter, boolean isNullable) {
	  super(requestFactory, callFactory, responseConverter);
	  this.callAdapter = callAdapter;
	  this.isNullable = isNullable;
	}

	@Override protected Object adapt(Call<ResponseT> call, Object[] args) {
	  // 参数里的 call 为 OkHttpCall
	  call = callAdapter.adapt(call);

	  //noinspection unchecked Checked by reflection inside RequestFactory.
	  // 获取最后一个参数 Continuation, 用于接收返回结果
	  Continuation<ResponseT> continuation = (Continuation<ResponseT>) args[args.length - 1];
	  // 这里不用纠结 isNullable, Retrofit 就没有实现如何判断方法返回结果是否
          // 可空，所以我们直接看 await 方法就行。
	  return isNullable
		  ? KotlinExtensions.awaitNullable(call, continuation)
		  : KotlinExtensions.await(call, continuation);
	}
}
```

```kotlin
suspend fun <T : Any> Call<T>.await(): T {
  return suspendCancellableCoroutine { continuation ->
    continuation.invokeOnCancellation {
      cancel()
    }
    // 执行 Call 的 enque 方法，里面具体的操作就是通过之前的 RequestFactory 创建
    // 的 OkHttp 的 Request, 再通过 Request 创建 OkHttp 的 Call, 最后执行 OkHttp 的
    // Call 的 enque 方法。使用了代理模式，具体的请求是交给 OkHttp 完成的。
    enqueue(object : Callback<T> {
      override fun onResponse(call: Call<T>, response: Response<T>) {
        // 判断请求是否成功
        if (response.isSuccessful) {
          val body = response.body()
          // 如果具体数据类型为 null 也表示失败，直接返回异常
          if (body == null) {
            val invocation = call.request().tag(Invocation::class.java)!!
            val method = invocation.method()
            val e = KotlinNullPointerException("Response from " +
                method.declaringClass.name +
                '.' +
                method.name +
                " was null but response body type was declared as non-null")
            // 返回异常
            // 所以执行 suspend 方法的时候我们需要 try catch 去捕获异常。
            continuation.resumeWithException(e)
          } else {
            // 返回具体数据类型
            continuation.resume(body)
          }
        } else {
          // 请求失败，返回异常
          // 所以执行 suspend 方法的时候我们需要 try catch 去捕获异常。
          continuation.resumeWithException(HttpException(response))
        }
      }

      override fun onFailure(call: Call<T>, t: Throwable) {
        // 请求失败，返回异常
        // 所以执行 suspend 方法的时候我们需要 try catch 去捕获异常。
        continuation.resumeWithException(t)
      }
    })
  }
}
```

## 小结

Retrofit 对协程的支持其实就增加了不到 200 行代码吧，我真很佩服 Square 的工程师们，
尤其是创建 suspend 方法的 CallAdapter 简直惊艳到我了，创建了一个自定义的 Call 类
型就完美地复用之前的代码，什么时候才能达到他们的高度呀？我想至少还有很长的路还有
很多的源码需要继续阅读，不断地了解他们解决问题思维，才能不断进步。

当我们了解了 Retrofit 是如何支持协程的，再开发过程中我们就能更好的解决遇到的问题，
最后也希望这篇文章对你来说有所收获。
