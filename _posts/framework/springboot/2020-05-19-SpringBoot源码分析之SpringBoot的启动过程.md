---
layout: post 
author: oshacker
title: SpringBoot源码分析之SpringBoot的启动过程
category: spring-all
tags: [java,spring-all]
excerpt: Spring全家桶：第二篇
---


Spring Boot的启动很简单，代码如下：
```java
@SpringBootApplication
public class HelloSpringApplication {
    public static void main(String[] args) {
        SpringApplication.run(HelloSpringApplication.class, args);
    }
}
```

从代码上可以看出，调用了SpringApplication的静态方法run，进入该run方法：
```java
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    return run(new Class<?>[] { primarySource }, args);
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return new SpringApplication(primarySources).run(args);
}
```

从上面的第二个run方法可知。它会构造一个SpringApplication的实例，然后再调用这个实例的run方法就表示启动Spring Boot。

因此，想要分析Spring Boot的启动过程，我们只需要分析SpringApplication的构造过程以及SpringApplication的run方法的执行过程即可。

## SpringApplication的构造过程

打开SpringApplication类，其构造方法如下：
```java
public SpringApplication(Class<?>... primarySources) {
    this(null, primarySources);
}

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    //把primarySources设置到SpringApplication的Set<Class<?>> primarySources属性中，目前Set中只有HelloSpringApplication.class
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //判断Web应用的类型：REACTIVE、NONE、SERVLET
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

其中，deduceFromClasspath()方法表示根据classpath下标志类的存在性推断Web应用的类型，具体实现是通过isPresent()尝试加载不同应用类型对应的标志类的字节码文件，代码如下：
```java
static WebApplicationType deduceFromClasspath() {
    //WebFlux的标志类org.springframework.web.reactive.DispatcherHandler存在于classpath下，但是WEBMVC的标志类
    //org.springframework.web.servlet.DispatcherServlet和JERSEY的标志类org.glassfish.jersey.servlet.ServletContainer
    //不在classpath下，则推断该Web应用是WebFlux类型
    if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
            && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
        return WebApplicationType.REACTIVE;
    }
    //推断出不是Spring WebFlux类型
    //Servlet的标志类javax.servlet.Servlet和org.springframework.web.context.ConfigurableWebApplicationContext，如果有
    //一个不在classpath下，则推断该应用不是Web应用；
    for (String className : SERVLET_INDICATOR_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
        }
    }
    //否则推断该Web应用是Servlet类型
    return WebApplicationType.SERVLET;
}
```

getSpringFactoriesInstances表示从类路径下 META-INF/spring.factories 下加载 SpringFactory 实例，类似的操作在 Dubbo SPI 中也有，代码如下：
```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
		return getSpringFactoriesInstances(type, new Class<?>[] {});
	}

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader(); //获取类加载器
    // Use names and ensure unique to protect against duplicates
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```


## SpringApplication中run方法的执行过程

## 总结

## 应用