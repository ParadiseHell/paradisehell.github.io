---
layout:     post
title:      "使用 Retrofit 如何丢弃烦人的 BaseResponse"
subtitle:   "再也不用写重复处理 BaseResponse 的代码了"
date:       2021-06-05
author:     "ChengTao"
header-img: "img/android.png"
tags:
    - Android
    - Java
    - 源码解析
    - 开源
---

> 由于后台返回统一数据结构，比如 code, data, message; 使用过 Retrofit
的同学一定定义过类似 BaseResponse 这种类，但是 BaseResponse 的处理逻辑都大同小异，
每次都写着实让人很烦，有没有什么好的方式解决这一痛点呢？本文讲介绍一种优雅的方式
来解决这一问题。

# 背景
当我们打开 Retrofit 的官方文档时，官方的例子是这样的：

```java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```

而到了我们自己的项目，大多情况确实这样的：

```java
public interface XXXService {
  @GET("xxx")
  Call<BaseResponse<XXX>> xxx();
}
```

而不同的公司，有不同的数据结构，不过都是大同小异，比如 `code, data, message` 或者
`status, data, message`, 每次都要写什么 `code == 0` 之类的代码，无聊不说，主要一点
含量都没有。。。

如果我们要是能把 BaseResponse 去掉，岂不美哉？就像下面的定义一样：

```java
public interface XXXService {
  @GET("xxx")
  Call<XXX> xxx();
}
```

如果是 Kotlin 就更爽了，直接 suspend 方法走起。

```kotlin
interface XXXService {
  @GET("xxx")
  suspend fun xxx() : XXX
}
```


# Convert.Factory 中被忽略的参数

```java
public interface Converter<F, T> {
  abstract class Factory {
    // 参数 annotations 是 Method 上的注解
    public @Nullable Converter<ResponseBody, ?> responseBodyConverter(
        Type type, Annotation[] annotations, Retrofit retrofit) {
      return null;
    }
  }
}

public final class GsonConverterFactory extends Converter.Factory {
  // Retrofit 官方的 Converter.Factory 并没有使用 annotations 这个参数
  @Override
  public Converter<ResponseBody, ?> responseBodyConverter(
      Type type, Annotation[] annotations, Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonResponseBodyConverter<>(gson, adapter);
  }
}
```
从上面的代码不难看出，在实现 Convert.Factory 的时候，
Retrofit 官方的实现并没有使用 annotations 这个参数，而这个参数恰恰是移除
BaseResponse 的关键。

# 实现方案

其实实现方案很简单，就是给 Method 再加一个自定义注解，然后对应实现一个
Converter.Factory, 再这个 Converter.Factory 类中判断 Method 是否存在我们自定义的
注解，如果有就将数据进行转换，最后交给 Gson 进行解析。

项目地址：[Convex](https://github.com/ParadiseHell/convex)

## 实现细节

数据转换接口：

```kotlin
interface ConvexTransformer {
    // 将原始的 InputStream 转换成具体业务数据的 InputStream
    // 相当于获取 code, data, message 的数据流转换成只有 data 的数据流
    @Throws(IOException::class)
    fun transform(original: InputStream): InputStream
}
```

自定义注解：

```kotlin
@Target(AnnotationTarget.CLASS, AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class Transformer(val value: KClass<out ConvexTransformer>)

// 有了这个自定义注解，我们就可以给 Method 添加这个注解了
interface XXXService {
    @GET("xxx")
    @Transformer(XXXTransformer::class)
    fun xxxMethod() : Any
}
```

Convert.Factory 实现类：

```kotlin
class ConvexConverterFactory : Converter.Factory() {
    override fun responseBodyConverter(
        type: Type,
        annotations: Array<Annotation>,
        retrofit: Retrofit
    ): Converter<ResponseBody, *>? {
        // 获取 Method 上的 Transformer 注解
        // 从而确定使用那个具体的 ConvexTransformer 类
        return annotations
            .filterIsInstance<Transformer>()
            .firstOrNull()?.value
            ?.let { transformerClazz ->
                // 获取可以处理当前返回值的 Converter
                // ConvexConverterFactory 只做数据流转，具体数据序列化
                // 交给类似 GsonConverterFactory
                retrofit.nextResponseBodyConverter<Any>(
                    this, type, annotations
                )?.let { converter ->
                    // 如果有能处理返回值的 Converter 创建代理的 ConvexConveter
                    ConvexConverter<Any>(transformerClazz, converter)
                }
            }
    }
}

class ConvexConverter<T> constructor(
    private val transformerClazz: KClass<out ConvexTransformer>,
    private val candidateConverter: Converter<ResponseBody, *>
) : Converter<ResponseBody, T?> {

    @Suppress("UNCHECKED_CAST")
    @Throws(IOException::class)
    override fun convert(value: ResponseBody): T? {
        // 从 Convex 中获取 ConvexTransformer
        // Convex 是一个 ServiceRegistry, 用于存储 ConvexTransformer
        // 如果没有 ConvexTransformer 便会通过反射创建对应的 ConvexTransformer
        // 并保存进 Convex 中等待下次使用
        return Convex.getConvexTransformer(transformerClazz.java)
            .let { transformer ->
                // 转换数据流，这里就是将 code, data, message 数据流转换成 data 数据流
                transformer.transform(value.byteStream()).let { responseStream ->
                    ResponseBody.create(value.contentType(), responseStream.readBytes())
                }
            }?.let { responseBody ->
                // 使用具体 Converter 将 data 数据流转换成具体的 data
                candidateConverter.convert(responseBody) as T?
            }
    }
}
```

# Convex 使用

## 添加依赖

```gralde
dependencies {
    implementation "org.paradisehell.convex:convex:1.0.0"
}
```

## 实现 ConvexTransformer

```kotlin
private class XXXConvexTransformer : ConvexTransformer {
	@Throws(IOException::class)
	override fun transform(original: InputStream): InputStream {
		TODO("Return the business data InputStream.")
	}
}
```

## 将 ConvexConverterFactory 加入 Retrofit 中

```kotlin
Retrofit.Builder()
	.baseUrl("https://xxx.com/")
	// ConvexConverterFactory 一定放在最前面
	.addConverterFactory(ConvexConverterFactory())
	.addConverterFactory(GsonConverterFactory.create())
	.build()
```

## 定义 Service 接口

```kotlin
interface XXXService {
	@GET("xxx")
    // 使用 Transformer 注解，添加具体的 ConvexTransformer 实现类
	@Transformer(XXXConvexTransformer::class)
	suspend fun xxx() : XXX
}
```

## 例子

- [ConvexTest](https://github.com/ParadiseHell/convex/blob/main/convex/src/test/kotlin/org/paradisehell/convex/ConvexTest.kt)
- [Android-Project](https://github.com/ParadiseHell/convex/blob/main/app/src/main/java/org/paradisehell/convex/MainActivity.kt)

## 更多

更多的使用方法和更便捷的使用方法可以具体看
[Convex](https://github.com/ParadiseHell/convex) 的 README.

# 小结

完美，以后定义 Method 再也不用写 BaseResponse 了，BaseResponse 终于可以统一处理了。

而且这种方案还支持多种不同的数据类型，因为不同的 Method 可以指定不同的
ConvexTransformer, 而到具体的业务处理根本不用关系 BaseResponse 是如何处理的，
因为具体的业务代码拿到的都是具体的 Data 数据。

不得不佩服 Retrofit 的设计，Converter.Factory 留出了 annotations 这个参数，可扩展
性简直强到爆，致敬 Square, Salute~~
