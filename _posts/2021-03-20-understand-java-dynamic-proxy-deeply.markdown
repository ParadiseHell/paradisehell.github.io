---
layout:     post
title:      "深入理解 Java 动态代理"
subtitle:   "知其然并知其所以然"
date:       2021-02-20
author:     "ChengTao"
header-img: "img/android.png"
tags:
    - Android
    - Java
    - 源码解析
---

> 开发安卓的小伙伴一定都用过 Retrofit, 那么一定也或多或少了解 Retrofit 的原理，
其中最重要的一个原理就是动态代理，那动态代理又是怎么实现的呢？这篇文章就将揭开
动态代理神秘的面纱。

# 动态代理快速上手

下面的例子将简单介绍如何使用动态代理：
```java
// 定义接口
public interface ProxyInterface {
	void a();

	...
}

// 使用动态代理创建对象
ProxyInterface impl = (ProxyInterface) Proxy.newProxyInstance(
	ProxyInterface.class.getClassLoader(),
	new Class[] {ProxyInterface.class},
	new InvocationHandler() {
	  @Override
	  public Object invoke(Object proxy, Method method, Object[] args)
		  throws Throwable {
		return null;
	  }
	});
impl.a();
```

# 动态代理源码解析
从上面的的例子我们可以看出，动态代理主要就是使用了 `Proxy#newInstance` 方法来创建
接口的代理实现类，那我们就从 `Proxy#newInstance` 方法入手，看看源码是如果实现的。

## Java JDK Proxy 类源码解析

**非 Android Java JDK !!!**

```java

// 代理类构造函数的参数类型
private static final Class<?>[] constructorParams = { InvocationHandler.class };


// Proxy 类的构造方法
protected Proxy(InvocationHandler h) {
	Objects.requireNonNull(h);
	this.h = h;
}

public static Object newProxyInstance(
	ClassLoader loader,
	Class<?>[] interfaces,
	InvocationHandler h) throws IllegalArgumentException {
	
	...

	// 查找或者通过 ClassLoader 和接口列表生成一个继承 Proxy 类并实现了接口列表
	// 的代理类的 Class
	Class<?> cl = getProxyClass0(loader, intfs);

	try {

		...	

		// 获取类型的构造方法
		final Constructor<?> cons = cl.getConstructor(constructorParams);

		...

		// 通过构造方法创建一个新的实例
		return cons.newInstance(new Object[]{h});
	} catch (IllegalAccessException|InstantiationException e) {
		throw new InternalError(e.toString(), e);
	} catch (InvocationTargetException e) {
		Throwable t = e.getCause();
		if (t instanceof RuntimeException) {
			throw (RuntimeException) t;
		} else {
			throw new InternalError(t.toString(), t);
		}
	} catch (NoSuchMethodException e) {
		throw new InternalError(e.toString(), e);
	}
}
```
从上面的源码我们可以简单的概括一下动态代理的原理：通过 ClassLoader 和接口列表生
成一个继承 Proxy 并实现了接口列表的代理类，然后通过反射创建一个代理类的实例。

从上面的源码中，我们不难看出 `getProxyClass0` 在整个动态代理起到了至关重要的作用，
所以我们继续看 `getProxyClass0` 是如何实现的：

```java
// 代理 Class 的缓存
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
    proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());

// Proxy#getProxyClass0
// 通过 ClassLoader 和接口列表生成一个继承 Proxy 并实现了接口列表的代理类
private static Class<?> getProxyClass0(
	ClassLoader loader,
	Class<?>... interfaces) {

	// 检查代理类需要实现的接口数量
	if (interfaces.length > 65535) {
		throw new IllegalArgumentException("interface limit exceeded");
	}

	// 如果代理类存在直接返回一个缓存的 copy,
	// 如果不存在通过 ProxyClassFactory 创建对应的代理类
	return proxyClassCache.get(loader, interfaces);
}

// WeakCache 中的成员变量
private final ReferenceQueue<K> refQueue
	= new ReferenceQueue<>();
// the key type is Object for supporting null key
private final ConcurrentMap<Object, ConcurrentMap<Object, Supplier<V>>> map
	= new ConcurrentHashMap<>();
private final ConcurrentMap<Supplier<V>, Boolean> reverseMap
	= new ConcurrentHashMap<>();
private final BiFunction<K, P, ?> subKeyFactory;
private final BiFunction<K, P, V> valueFactory;

// WeakCache 的构造函数
public WeakCache(BiFunction<K, P, ?> subKeyFactory, BiFunction<K, P, V> valueFactory) {
	// subKekFactory 是 Proxy 中的 KeyFactory
	this.subKeyFactory = Objects.requireNonNull(subKeyFactory);
	// valueFactory 是 Proxy 汇总的 ProxyClassFactory
	this.valueFactory = Objects.requireNonNull(valueFactory);
}

// WeakCache#get
// 从 cache 中获取代理，如果没有缓存择创建代理类
// key : ClassLoader
// parameters : 动态代理的接口列表
public V get(K key, P parameter) {
	Objects.requireNonNull(parameter);

	expungeStaleEntries();

	// 通过 ClassLoader 创建 key
	Object cacheKey = CacheKey.valueOf(key, refQueue);

	// 如果不存在对应的 Key，则创建给 key 创建对应的 valueMap
	// 这里 ValueMap 其实就是的 value 就是我们的的代理类
	ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
	if (valuesMap == null) {
		ConcurrentMap<Object, Supplier<V>> oldValuesMap
			= map.putIfAbsent(cacheKey,
							  valuesMap = new ConcurrentHashMap<>());
		if (oldValuesMap != null) {
			valuesMap = oldValuesMap;
		}
	}

	// 通过 key 和 parameter 创建 valueMap，相当于通过 ClassLoader 和接口列表
	// 生成一个唯一的 subKey
	Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
	// 通过 subKey 获取 Supplier
	Supplier<V> supplier = valuesMap.get(subKey);
	Factory factory = null;

	while (true) {
		// 如果 supplier 存在，直接获取 value 并返回
		if (supplier != null) {
			// 通过 supplier 获取代理类，这个是最重要的代码！！！
			V value = supplier.get();
			if (value != null) {
				return value;
			}
		}
		// 如果 supplier 不存在
		// 或者 supplier#get 返回了 null（可能因为被清空的 Cache 或者 Factory
		// 创建不成功）

		// 创建 Factory
		if (factory == null) {
			// key  是 ClassLoader
			// paramters  是于接口列表
			// subKey 相当于 ClassLoader 和接口列表生成的唯一 key
			factory = new Factory(key, parameter, subKey, valuesMap);
		}

		if (supplier == null) {
			// 将 Factory 存入 Map 中并将返回值赋值 supplier
			supplier = valuesMap.putIfAbsent(subKey, factory);
			if (supplier == null) {
				// 如果 supplier 为 null，证明未曾保存 Factory
				supplier = factory;
			}
		} else {
			// 替换 factory
			if (valuesMap.replace(subKey, supplier, factory)) {
				// 替换成功，赋值 suppiler
				// 成功是因为被清空的 CacheEntry 或者原来的 Factory 创建不成功
				supplier = factory;
			} else {
				// 替换不成功，从 valuesMap 再此获取 suppiler, 进行再次尝试
				supplier = valuesMap.get(subKey);
			}
		}
	}
}

// 上面的源码我们知道代理类是通过 suppiler 创建的，而实际起作用的是 Factory,
// 所以我们再看看 Factory 的源码。

private final class Factory implements Supplier<V> {

	private final K key;
	private final P parameter;
	private final Object subKey;
	private final ConcurrentMap<Object, Supplier<V>> valuesMap;

	// key  是 ClassLoader
	// paramters  是于接口列表
	// subKey 相当于 ClassLoader 和接口列表生成的唯一 key
	Factory(K key, P parameter, Object subKey,
			ConcurrentMap<Object, Supplier<V>> valuesMap) {
		this.key = key;
		this.parameter = parameter;
		this.subKey = subKey;
		this.valuesMap = valuesMap;
	}

	@Override
	public synchronized V get() { // serialize access
		...

		V value = null;
		try {
			// 通过 valueFactory 创建代理类
			// valueFactory 则是 Proxy 类中的 ProxyClassFactory
			// key 相当于 ClassLoader
			// parameter 是要代理的接口列表
			value = Objects.requireNonNull(valueFactory.apply(key, parameter));
		} finally {
			if (value == null) { // remove us on failure
				valuesMap.remove(subKey, this);
			}
		}

		...

		return value;
	}
}

// 兜兜转转我们又回到了 Proxy 类，我们继续看看 ProxyClasFactory 是怎么实现的

private static final class ProxyClassFactory
	implements BiFunction<ClassLoader, Class<?>[], Class<?>>
{
	// prefix for all proxy class names
	private static final String proxyClassNamePrefix = "$Proxy";

	// next number to use for generation of unique proxy class names
	private static final AtomicLong nextUniqueNumber = new AtomicLong();

	@Override
	public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
		
		...
		
		// 通过接口列表确认代理类的包名
		String proxyPkg = null;     // package to define proxy class in
		int accessFlags = Modifier.PUBLIC | Modifier.FINAL;
		for (Class<?> intf : interfaces) {
			int flags = intf.getModifiers();
			if (!Modifier.isPublic(flags)) {
				accessFlags = Modifier.FINAL;
				String name = intf.getName();
				int n = name.lastIndexOf('.');
				String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
				if (proxyPkg == null) {
					proxyPkg = pkg;
				} else if (!pkg.equals(proxyPkg)) {
					throw new IllegalArgumentException(
						"non-public interfaces from different packages");
				}
			}
		}
		if (proxyPkg == null) {
			// if no non-public proxy interfaces, use com.sun.proxy package
			proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
		}

		// 生成唯一的代理类名称
		long num = nextUniqueNumber.getAndIncrement();
		String proxyName = proxyPkg + proxyClassNamePrefix + num;

		// 通过 ProxyGenerator 创建代理类的字节码！！！
		byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
			proxyName, interfaces, accessFlags);
		try {
			// 通过字节码定义代理类！！！
			// defineClass0 是 native 方法，这里就不展开了，能力有限
			// 不过这个就很像 ClassLoader 的 defineClass 方法
			return defineClass0(loader, proxyName,
								proxyClassFile, 0, proxyClassFile.length);
		} catch (ClassFormatError e) {
			throw new IllegalArgumentException(e.toString());
		}
	}
}
```
到此为止，我们便可以更全面的概括动态代理的原理，通过接口列表生成继承 Proxy 类并
实现了代理接口的字节码，然后通过类似 ClassLoader#defineClass 方法定义代理类的
Class, 最后通过反射创建代理类实例。

## 揭开代理类字节码的庐山真面目

如何获取代理类的字节码呢？从上面的源码分析中我们不难看出，
`ProxyGenerator#generateProxyClass` 是生成字节码的关键，那我们就仿造源码中的写法，
生成一个代理类的字节码看看，具体代码如下：

```java
public class ProxyDemo {

  public interface ProxyInterface {
    void a();

    void b(String s);

    String c();

    String d(String s);
  }

  public static void main(String[] args) {
    byte[] byteCode = ProxyGenerator.generateProxyClass(
        "ProxyInterfaceImpl", new Class[] {ProxyInterface.class}
    );
    FileOutputStream out = null;
    try {
      out = new FileOutputStream("ProxyInterfaceImpl.class");
      out.write(byteCode);
      out.flush();
      System.out.println("Success");
    } catch (Exception e) {
      e.printStackTrace();
    } finally {
      try {
        if (out != null) {
          out.close();
        }
      } catch (IOException e) {
        e.printStackTrace();
      }
    }
  }
}
```

我们将字节码保存在了一个 ProxyInterfaceImpl.class 的文件中，那我我们就可以使用
javap 命令反汇编看看生成的字节码到底是什么样子的。

```shell
javap -p ProxyInterfaceImpl
```

下面代理类的所有成员变量：

```txt
public final class ProxyInterfaceImpl extends java.lang.reflect.Proxy implements org.paradisehell.test.ProxyDemo$ProxyInterface {
  private static java.lang.reflect.Method m1;
  private static java.lang.reflect.Method m4;
  private static java.lang.reflect.Method m6;
  private static java.lang.reflect.Method m3;
  private static java.lang.reflect.Method m5;
  private static java.lang.reflect.Method m2;
  private static java.lang.reflect.Method m0;
  
  ...
}
```

可以看出代理类有 6 个 Method 类型的成员变量，不过并看不出这几个方法是什么意思，
那我们反汇编代码看看。

```shell
javap -c ProxyInterfaceImpl
```

下面是代理类代码反汇编的结果：

```txt
public final class ProxyInterfaceImpl extends java.lang.reflect.Proxy implements org.paradisehell.test.ProxyDemo$ProxyInterface {
  // 构造方法
  public ProxyInterfaceImpl(java.lang.reflect.InvocationHandler) throws ;
    Code:
       0: aload_0
       1: aload_1
       2: invokespecial #8                  // Method java/lang/reflect/Proxy."<init>":(Ljava/lang/reflect/InvocationHandler;)V
       5: return
  // 翻译后的 Java 代码
  publick ProxyInterfaceImpl(InvocationHandler h) {
    super.(h) 
  }
  
  // equals 方法，对应 Object 中的 equals 方法
  public final boolean equals(java.lang.Object) throws ;
    Code:
       0: aload_0
       1: getfield      #16                 // Field java/lang/reflect/Proxy.h:Ljava/lang/reflect/InvocationHandler;
       4: aload_0
       5: getstatic     #20                 // Field m1:Ljava/lang/reflect/Method;
       8: iconst_1
       9: anewarray     #22                 // class java/lang/Object
      12: dup
      13: iconst_0
      14: aload_1
      15: aastore
      16: invokeinterface #28,  4           // InterfaceMethod java/lang/reflect/InvocationHandler.invoke:(Ljava/lang/Object;Ljava/lang/reflect/Method;[Ljava/lang/Object;)Ljava/lang/Object;
      21: checkcast     #30                 // class java/lang/Boolean
      24: invokevirtual #34                 // Method java/lang/Boolean.booleanValue:()Z
      27: ireturn
      28: athrow
      29: astore_2
      30: new           #42                 // class java/lang/reflect/UndeclaredThrowableException
      33: dup
      34: aload_2
      35: invokespecial #45                 // Method java/lang/reflect/UndeclaredThrowableException."<init>":(Ljava/lang/Throwable;)V
      38: athrow
    Exception table:
       from    to  target type
           0    28    28   Class java/lang/Error
           0    28    28   Class java/lang/RuntimeException
           0    28    29   Class java/lang/Throwable
  // 翻译后的 Java 代码
  public final boolean equals(Object obj) {
    try {
      h.invoke(this, m1, new Object[]{ obj });
    } catch(Thowable t) {
      throw new UndeclaredThrowableException(t.getMessage());
    }
  }

  // 接口中定义的 d 方法
  public final java.lang.String d(java.lang.String) throws ;
    Code:
       0: aload_0
       1: getfield      #16                 // Field java/lang/reflect/Proxy.h:Ljava/lang/reflect/InvocationHandler;
       4: aload_0
       5: getstatic     #50                 // Field m4:Ljava/lang/reflect/Method;
       8: iconst_1
       9: anewarray     #22                 // class java/lang/Object
      12: dup
      13: iconst_0
      14: aload_1
      15: aastore
      16: invokeinterface #28,  4           // InterfaceMethod java/lang/reflect/InvocationHandler.invoke:(Ljava/lang/Object;Ljava/lang/reflect/Method;[Ljava/lang/Object;)Ljava/lang/Object;
      21: checkcast     #52                 // class java/lang/String
      24: areturn
      25: athrow
      26: astore_2
      27: new           #42                 // class java/lang/reflect/UndeclaredThrowableException
      30: dup
      31: aload_2
      32: invokespecial #45                 // Method java/lang/reflect/UndeclaredThrowableException."<init>":(Ljava/lang/Throwable;)V
      35: athrow
    Exception table:
       from    to  target type
           0    25    25   Class java/lang/Error
           0    25    25   Class java/lang/RuntimeException
           0    25    26   Class java/lang/Throwable
  // 翻译后的 Java 代码
  public final String d(String s {
    try {
      h.invoke(this, m4, new Object[]{ s });
    } catch(Thowable t) {
      throw new UndeclaredThrowableException(t.getMessage());
    }
  }

  // 其他方法执行也是类似的，就是获取 Proxy 中的 InvocationHanlder, 然后找到当前
  // 方法对应

  // 接口中定义的 b 方法，其中执行的方法为 m6
  ...

  // 接口中定义的 a 方法，其中执行的方法为 m3
  ...

  // 接口中定义的 c 方法，其中执行的方法为 m5
  ...

  // Object 中的 toString 方法，其中执行的方法为 m2
  ...

  // Object 中的 hasCode 方法，其中执行的方法为 m0
  ...

  // static 代码块
  static {} throws ;
    Code:
       0: ldc           #84                 // String java.lang.Object
       2: invokestatic  #90                 // Method java/lang/Class.forName:(Ljava/lang/String;)Ljava/lang/Class;
       5: ldc           #91                 // String equals
       7: iconst_1
       8: anewarray     #86                 // class java/lang/Class
      11: dup
      12: iconst_0
      13: ldc           #84                 // String java.lang.Object
      15: invokestatic  #90                 // Method java/lang/Class.forName:(Ljava/lang/String;)Ljava/lang/Class;
      18: aastore
      19: invokevirtual #95                 // Method java/lang/Class.getMethod:(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
      22: putstatic     #20                 // Field m1:Ljava/lang/reflect/Method;
      25: ldc           #97                 // String org.paradisehell.test.ProxyDemo$ProxyInterface
      27: invokestatic  #90                 // Method java/lang/Class.forName:(Ljava/lang/String;)Ljava/lang/Class;
      30: ldc           #98                 // String d
      32: iconst_1
      33: anewarray     #86                 // class java/lang/Class
      36: dup
      37: iconst_0
      38: ldc           #100                // String java.lang.String
      40: invokestatic  #90                 // Method java/lang/Class.forName:(Ljava/lang/String;)Ljava/lang/Class;
      43: aastore
      44: invokevirtual #95                 // Method java/lang/Class.getMethod:(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
      47: putstatic     #50                 // Field m4:Ljava/lang/reflect/Method;
      50: ldc           #97                 // String org.paradisehell.test.ProxyDemo$ProxyInterface
      52: invokestatic  #90                 // Method java/lang/Class.forName:(Ljava/lang/String;)Ljava/lang/Class;
      55: ldc           #101                // String b
      57: iconst_1
      58: anewarray     #86                 // class java/lang/Class
      61: dup
      62: iconst_0
      63: ldc           #100                // String java.lang.String
      65: invokestatic  #90                 // Method java/lang/Class.forName:(Ljava/lang/String;)Ljava/lang/Class;
      68: aastore
      69: invokevirtual #95                 // Method java/lang/Class.getMethod:(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
      72: putstatic     #57                 // Field m6:Ljava/lang/reflect/Method;
      75: ldc           #97                 // String org.paradisehell.test.ProxyDemo$ProxyInterface
      77: invokestatic  #90                 // Method java/lang/Class.forName:(Ljava/lang/String;)Ljava/lang/Class;
      80: ldc           #102                // String a
      82: iconst_0
      83: anewarray     #86                 // class java/lang/Class
      86: invokevirtual #95                 // Method java/lang/Class.getMethod:(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
      89: putstatic     #62                 // Field m3:Ljava/lang/reflect/Method;
      92: ldc           #97                 // String org.paradisehell.test.ProxyDemo$ProxyInterface
      94: invokestatic  #90                 // Method java/lang/Class.forName:(Ljava/lang/String;)Ljava/lang/Class;
      97: ldc           #103                // String c
      99: iconst_0
     100: anewarray     #86                 // class java/lang/Class
     103: invokevirtual #95                 // Method java/lang/Class.getMethod:(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
     106: putstatic     #67                 // Field m5:Ljava/lang/reflect/Method;
     109: ldc           #84                 // String java.lang.Object
     111: invokestatic  #90                 // Method java/lang/Class.forName:(Ljava/lang/String;)Ljava/lang/Class;
     114: ldc           #104                // String toString
     116: iconst_0
     117: anewarray     #86                 // class java/lang/Class
     120: invokevirtual #95                 // Method java/lang/Class.getMethod:(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
     123: putstatic     #71                 // Field m2:Ljava/lang/reflect/Method;
     126: ldc           #84                 // String java.lang.Object
     128: invokestatic  #90                 // Method java/lang/Class.forName:(Ljava/lang/String;)Ljava/lang/Class;
     131: ldc           #105                // String hashCode
     133: iconst_0
     134: anewarray     #86                 // class java/lang/Class
     137: invokevirtual #95                 // Method java/lang/Class.getMethod:(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
     140: putstatic     #76                 // Field m0:Ljava/lang/reflect/Method;
     143: return
     144: astore_1
     145: new           #109                // class java/lang/NoSuchMethodError
     148: dup
     149: aload_1
     150: invokevirtual #112                // Method java/lang/Throwable.getMessage:()Ljava/lang/String;
     153: invokespecial #114                // Method java/lang/NoSuchMethodError."<init>":(Ljava/lang/String;)V
     156: athrow
     157: astore_1
     158: new           #118                // class java/lang/NoClassDefFoundError
     161: dup
     162: aload_1
     163: invokevirtual #112                // Method java/lang/Throwable.getMessage:()Ljava/lang/String;
     166: invokespecial #119                // Method java/lang/NoClassDefFoundError."<init>":(Ljava/lang/String;)V
     169: athrow
    Exception table:
       from    to  target type
           0   144   144   Class java/lang/NoSuchMethodException
           0   144   157   Class java/lang/ClassNotFoundException
  // 翻译后的 Java 代码
  static {
    try {
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[]{ Object.class });
      m4 = Class.forName("org.paradisehell.test.ProxyDemo$ProxyInterface").getMethod("d", new Class[]{ String.class });
      m6 = Class.forName("org.paradisehell.test.ProxyDemo$ProxyInterface").getMethod("b", new Class[]{ String.class });
      m3 = Class.forName("org.paradisehell.test.ProxyDemo$ProxyInterface").getMethod("a", new Class[]{});
      m5 = Class.forName("org.paradisehell.test.ProxyDemo$ProxyInterface").getMethod("c", new Class[]{});
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[]{});
      m0 = Class.forName("java.lang.Object").getMethod("hasCode", new Class[]{});
    } catch(NoSuchMethodError e1) {
      throw new NoSuchMethodError(e1.getMessage()); 
    } catch(NoClassDefFoundError e2) {
      throw new NoClassDefFoundError(e2.getMessage()); 
    }
  } 
}
```
这下就清晰多了，通过上面的反汇编我们可以得到下面几个代理类的特征：
- 继承 Proxy
- 实现了接口方法
- 实现了 Object 的 equals, toString 和 hashCode 三个方法

最后，代理原理也很简单，就是在 static 块通过反射获取要代理的方法并保存在成员变量
中，然后在调用相关方法的时候，找到保存在成员变量中的对应的方法通过 Proxy 的
InvocationHandler 执行其 invoke 方法就完成了代理。

## Android JDK Proxy 类源码解析

Android JDK Proxy 的源码和 Java JDK Proxy 的源码大同小异，主要区别在于
ProxyClassFactory 的实现：
```java
private static final class ProxyClassFactory
	implements BiFunction<ClassLoader, Class<?>[], Class<?>>
{
	// prefix for all proxy class names
	private static final String proxyClassNamePrefix = "$Proxy";

	// next number to use for generation of unique proxy class names
	private static final AtomicLong nextUniqueNumber = new AtomicLong();

	@Override
	public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

        ...
        
        // 获取代理类的包名
		String proxyPkg = null;     // package to define proxy class in
		int accessFlags = Modifier.PUBLIC | Modifier.FINAL;
		for (Class<?> intf : interfaces) {
			int flags = intf.getModifiers();
			if (!Modifier.isPublic(flags)) {
				accessFlags = Modifier.FINAL;
				String name = intf.getName();
				int n = name.lastIndexOf('.');
				String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
				if (proxyPkg == null) {
					proxyPkg = pkg;
				} else if (!pkg.equals(proxyPkg)) {
					throw new IllegalArgumentException(
						"non-public interfaces from different packages");
				}
			}
		}
		if (proxyPkg == null) {
			// if no non-public proxy interfaces, use the default package.
			proxyPkg = "";
		}

		{
            // Android 这里没有通过 ProxyGenerator 生成字节码
            // 而是获取要代理的方法，直接生成代理类的 Class
			List<Method> methods = getMethods(interfaces);
			Collections.sort(methods, ORDER_BY_SIGNATURE_AND_SUBTYPE);
			validateReturnTypes(methods);
			List<Class<?>[]> exceptions = deduplicateAndGetExceptions(methods);

			Method[] methodsArray = methods.toArray(new Method[methods.size()]);
			Class<?>[][] exceptionsArray = exceptions.toArray(new Class<?>[exceptions.size()][]);

			/*
			 * Choose a name for the proxy class to generate.
			 */
			long num = nextUniqueNumber.getAndIncrement();
			String proxyName = proxyPkg + proxyClassNamePrefix + num;

            // generateProxy 为 native 方法并且方法的返回值为 class
            // 参数分别是：
            // proxyName : 代理类的名称（含包名）
            // interfaces : 需要实现的接口列表
            // loader : ClassLoader
            // methodArray : 存有要代理的方法数组
            // exceptionsArray: 存有异常的数组，长度和 methodArray 相等，并和方法一一对应
			return generateProxy(proxyName, interfaces, loader, methodsArray,
								 exceptionsArray);
		}
	}
}
```
关于 `gernrateProxy` 是如何实现的，有兴趣的同学可以看 AOSP 的源码，这里就不展开
讲了（其实我看不懂，哈哈哈）。

下面的这两个链接是 `gernrateProxy` 源码的实现代码，有兴趣的同学可以看一下
（需要翻墙）。
- [java_lang_reflect_Proxy.cc](https://cs.android.com/android/platform/superproject/+/master:art/runtime/native/java_lang_reflect_Proxy.cc;l=32?q=generateProxy&ss=android)
- [class_linker.cc](https://cs.android.com/android/platform/superproject/+/master:art/runtime/class_linker.cc;drc=master;l=4848)

不过可以告诉大家一个结论，C++ 代码里并没有生成对应的字节码，而是通过 C++ 中的
class 类直接设置对应的方法、接口列表、ClassLoader 等一系列参数，直接生成了对应
的代理类. 如果这里我理解有误还请大佬指出来，这里先谢谢了。

除此之外，Android 的代理类，也有类似的 m0,m1,m2,m3 的方法，大家可以自己写个例子，
通过反射获取动态代理的实例获取代理类相关的属性和方法，和 Java 版本没什么区别，只
不过 Android 生成代理类的过程都放在了 native 层，少了和 Java 层的再次交互，速度
上更快一些，我想这也是 Android JDK Proxy 的 `generateProxy` 方法加了 `FastNative`
注解的原因吧。

# 总结

动态代理的原理：Java 版本通过生成继承了 Proxy 并实现了接口列表的对应代理类的字节码，
然后通过 defineClass 方式定义对应的代理类, 最后通过反射创建代理对象实例；而
Android 版本直接通过方法、接口列表、ClassLoader 等参数直接生成了对应代理类，
省去了生成字节码的过程，将代理类的生成都放在了 navtive 层。

代理类的方法执行原理：代理类在 static 块通过放射讲要代理的方法保存在成员变量中，
具体方法执行的时候获取对应的方法通过 Proxy 的 InvocationHandler 的 invoke 方法
代理具体的方法，这里的 InvocationHandler 其实就是我们 `Proxy#newInstance` 传入的
InvocationHandler。

# 写在最后
使用了 Retrofit 很多年了，也知道它使用了动态代理的设计模式，不过对动态代理的原理
其实是一知半解，通过这次源码分析，算是比较彻底的搞明白的动态代理的原理，不过一个
挺要命的东西就是一碰到 native 方法就傻眼，希望在以后工作和学习中能有机会好好的研
究一下 C++ 吧，这样才算真的彻底弄明白了源码原理。最后希望这篇文章可以帮助正在学习
或者对动态代理感兴趣的你。
