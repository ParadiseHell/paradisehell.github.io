---
layout:     post
title:      "Retrofit + Kotlin-Coroutines 如何去掉 try catch"
subtitle:   "代码进一步简洁"
date:       2021-12-05
author:     "ChengTao"
header-img: "img/android.png"
tags:
    - Android
    - Java
    - 开源
---

# 背景
Retrofit 2.6.0 版本后对 suspend 方法进行了支持，对使用 kotlin 的开发者来说简直是福音，
但是执行 suspend 方法的时候异常处理仍然是件繁琐的事情，必须显示的执行 try catch,
或者使用 kotlin 自带的异常处理类 `CoroutineExceptionHandler` 进行处理，但是不管那种方式，
代码都很挫，不够优雅。

# 优雅的代码

```kotlin
val service = retrofit.create(WanAndroidService::class.java).proxyRetrofit()
// 执行 test
service.test()
	.onSuccess { println("execute test success ==> $it") }
	.onFailure { println("execute test() failure ==> $it") }
	// 执行 userInfo
	.onFailureThen { service.userInfo() }
	?.onSuccess { println("execute userInfo success ==> $it") }
	?.onFailure { println("execute userInfo() failure ==> $it") }
	// 执行 banner
	?.onFailureThen { service.banner() }
	?.onSuccess { println("execute banner() success ==> $it") }
	?.onFailure { println("execute banner() failure ==> $it") }
```
**没有任何的 try catch !!!**

运行结果如下：

```txt
execute test() failure ==> Failure(code=-1, message=HTTP 404 Not Found)
execute userInfo() failure ==> Failure(code=-1001, message=请先登录！)
execute banner() success ==> [{"desc":"一起来做个App吧","id":10,"imagePath":"https://www.wanandroid.com/blogimgs/50c115c2-cf6c-4802-aa7b-a4334de444cd.png","isVisible":1,"order":1,"title":"一起来做个App吧","type":0,"url":"https://www.wanandroid.com/blog/show/2"},{"desc":"","id":6,"imagePath":"https://www.wanandroid.com/blogimgs/62c1bd68-b5f3-4a3c-a649-7ca8c7dfabe6.png","isVisible":1,"order":1,"title":"我们新增了一个常用导航Tab~","type":1,"url":"https://www.wanandroid.com/navi"},{"desc":"","id":20,"imagePath":"https://www.wanandroid.com/blogimgs/90c6cc12-742e-4c9f-b318-b912f163b8d0.png","isVisible":1,"order":2,"title":"flutter 中文社区 ","type":1,"url":"https://flutter.cn/"}]
```

如果读到这里，你觉得这个代码不是你想象中的优雅的代码，那么你可以关掉当前网页了。

# 实现原理

从上面的例子，主要起作用的是 `proxyRetrofit` 方法，其他的 `onSuccess`, `onFailure` 和
`onFailureThen` 不过是扩展方法而已，那我们就一起来看一下 `proxyRetrofit` 的实现原理。

## Retrofit 处理 suspend 方法原理

在了解如何去掉 `try catch` 前我们需要先了解一下 retrofit 是如何处理 suspend 方法的，
具体实现原理可以参考 [Retrofit 如何处理协程](https://paradisehell.org/2021/01/30/how-do-retorfit-handle-coroutines/)。

这里我们注重看下面这段代码：

```kotlin
suspend fun <T : Any> Call<T>.await(): T {
  return suspendCancellableCoroutine { continuation ->
    continuation.invokeOnCancellation {
      cancel()
    }
    enqueue(object : Callback<T> {
      override fun onResponse(call: Call<T>, response: Response<T>) {
        if (response.isSuccessful) {
          val body = response.body()
          if (body == null) {
            val invocation = call.request().tag(Invocation::class.java)!!
            val method = invocation.method()
            val e = KotlinNullPointerException("Response from " +
                method.declaringClass.name +
                '.' +
                method.name +
                " was null but response body type was declared as non-null")
            // 请求失败
            continuation.resumeWithException(e)
          } else {
            // 请求成功
            continuation.resume(body)
          }
        } else {
          continuation.resumeWithException(HttpException(response))
        }
      }

      override fun onFailure(call: Call<T>, t: Throwable) {
        // 请求失败
        continuation.resumeWithException(t)
      }
    })
  }
}

// resume 方法会返回 success
public inline fun <T> Continuation<T>.resume(value: T): Unit =
    resumeWith(Result.success(value))

// resumeWithException 方法会返回 failure
public inline fun <T> Continuation<T>.resumeWithException(exception: Throwable): Unit =
    resumeWith(Result.failure(exception))
```

从 retrofit 的源码可以得知，导致运行时会抛出异常的罪魁祸首是 `resumeWithException`
导致的，那么如果我们能拦截 `resumeWithException` 方法则可以避免异常的抛出。

## 拦截 resumeWithException 原理

``` kotlin
/**
* ThrowableResolver 的作用是在运行时遇到异常后如果处理异常
*/
interface ThrowableResolver<T> {
    // 处理异常，并返回一个处理后的数据
    fun resolve(throwable: Throwable): T
}

/**
* `proxyRetrofit` 方法主要作用是重新对接口进行动态代理，这样就可以在
* `InvocationHandler#invoke` 中对异常进行拦截，这样调用方就不用显示地调用
* `try catch` 了。
*/
inline fun <reified T> T.proxyRetrofit(): T {
    // 获取原先的 retrofit 的代理对象的的 InvocationHandler
    // 这样我就可以继续使用 retrofit 的能力进行网络请求
    val retrofitHandler = Proxy.getInvocationHandler(this)
    return Proxy.newProxyInstance(
        T::class.java.classLoader, arrayOf(T::class.java)
    ) { proxy, method, args ->
        // 判断当前是为 suspend 方法
        method.takeIf { it.isSuspendMethod }?.getSuspendReturnType()
            // 通过方法的返回值获取一个 ThrowableResolver 处理异常
            ?.let { FactoryRegistry.getThrowableResolver(it) }
            ?.let { resolver ->
                // 替换原始的 Contiuation 对象，这样我们就可以对异常进行拦截
                args.updateAt(
                    args.lastIndex,
                    FakeSuccessContinuationWrapper(
                        args.last() as Continuation<Any>,
                        resolver as ThrowableResolver<Any>
                    )
                )
            }
        retrofitHandler.invoke(proxy, method, args)
    } as T
}

/**
 * 给 Method 添加的一个扩展属性，判断当前方法是不是 suspend 方法
 */
val Method.isSuspendMethod: Boolean
    get() = genericParameterTypes.lastOrNull()
        ?.let { it as? ParameterizedType }?.rawType == Continuation::class.java

/**
 * 给 Method 添加的扩展方法，获取当前 suspend 方法的返回值类型
 */
fun Method.getSuspendReturnType(): Type? {
    return genericParameterTypes.lastOrNull()
        ?.let { it as? ParameterizedType }?.actualTypeArguments?.firstOrNull()
        ?.let { it as? WildcardType }?.lowerBounds?.firstOrNull()
}

/**
 * Array 的扩展方法，更新指定 index 的值
 */
fun Array<Any?>.updateAt(index: Int, updated: Any?) {
    this[index] = updated
}

/**
 * Continuation 包装类，通过返回一个假的 Success（里面包含异常信息）拦截异常抛出
 */
class FakeSuccessContinuationWrapper<T>(
    private val original: Continuation<T>,
    private val throwableResolver: ThrowableResolver<T>,
) : Continuation<T> {

    override val context: CoroutineContext = original.context

    override fun resumeWith(result: Result<T>) {
        // 判断 result 是否为 success
        result.onSuccess {
            // 如果为 success 直接返回
            original.resumeWith(result)
        }.onFailure {
            // 如果为 failure, 返回一个假的 success, 这样协程判断当前为 success
            // 就不会抛出异常了，这样我们就达到了拦截异常的作用
            val fakeSuccessResult = throwableResolver.resolve(it)
            original.resumeWith(Result.success(fakeSuccessResult))
        }
    }
}
```

到这里我们就揭开了拦截协程异常的原理，通过包装原始的 Continuation, 在 result 为
failure 的时候，返回一个假的 success, 则协程就不会抛出异常。

## 更多例子

更详细的代码例子可以看 [one](https://github.com/ParadiseHell/one) 这个项目，更具体可以看
[OneTest](https://github.com/ParadiseHell/one/blob/main/src/test/kotlin/org/paradisehell/one/OneTest.kt)
这个测试用例。

**one** 除了展示了如果拦截 `try catch`, 还展示了如何统一处理不同的 Response 相应，
转换成一样的数据结构，具体参考 [使用 Retrofit 如何丢弃烦人的 BaseResponse](https://paradisehell.org/2021/06/05/get-rid-of-base-response-using-retrofit/)。

# 小结

通过替换 suspend 方法的 `Continuation` 可以完成 `try catch` 的拦截，再给项目中的
`BaseResponse` 添加一些扩展方法（建议仿照 Kotlin 的 [Result](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-result/)
API），则可以让我们网络请求变得无比简洁。

但是这一切的前提都是 Retrofit 以及它动态代理的思想，所以 Retrofit yyds.
