<center><h1>slf4j 日志框架实现原理</h1></center>

slf4j是门面模式的典型应用，先回顾一下门面模式。

门面模式，其核心为外部与一个子系统的通信必须通过一个统一的外观对象进行，使得子系统更易于使用.

  门面模式的核心为Facade即门面对象，门面对象核心为几个点：

- 知道所有子角色的功能和责任
- 将客户端发来的请求委派到子系统中，没有实际业务逻辑
- 不参与子系统内业务逻辑的实现



<h2>为什么要使用slf4j</h2>

项目里使用了logback这个日志系统，但是项目中依赖了其他jar包，这些jar包使用的日志系统log4j，slf4j-simple。这样的话，一个项目系统中使用了不同的日志框架，非常的不方便。而slf4j屏蔽了这三种日志系统的，与我们直接打交道的是sfl4j-api里的LoggerFactory.getLogger。slf4j是日志标准，并不是实现，logback,log4j,slf4j-simple是slf4j的具体实现。

slf4j源码分析LoggerFactory.getLogger：

```java
private final static Logger logger = LoggerFactory.getLogger(KafKaProducerService.class);
```

getLooger()如下：

```java
public static Logger getLogger(Class<?> clazz) {
    Logger logger = getLogger(clazz.getName());
    if (DETECT_LOGGER_NAME_MISMATCH) {
        Class<?> autoComputedCallingClass = Util.getCallingClass();
        if (autoComputedCallingClass != null && nonMatchingClasses(clazz, autoComputedCallingClass)) {
            Util.report(String.format("Detected logger name mismatch. Given name: \"%s\"; computed name: \"%s\".", logger.getName(),
                            autoComputedCallingClass.getName()));
            Util.report("See " + LOGGER_NAME_MISMATCH_URL + " for an explanation");
        }
    }
    return logger;
}
```

```java
public static Logger getLogger(String name) {
    ILoggerFactory iLoggerFactory = getILoggerFactory();
    return iLoggerFactory.getLogger(name);
}
```

getILoggerFactory()就是获取具体日志实例工厂:

```java
public static ILoggerFactory getILoggerFactory() {
    //INITIALIZATION_STATE是日志实例状态标识，UNINITIALIZED表示还没有实例
    if (INITIALIZATION_STATE == UNINITIALIZED) {
        //ONGOING_INITIALIZATION正在实例化
        INITIALIZATION_STATE = ONGOING_INITIALIZATION;
        performInitialization();
    }
    switch (INITIALIZATION_STATE) {
    case SUCCESSFUL_INITIALIZATION: //已经实例化过
        return StaticLoggerBinder.getSingleton().getLoggerFactory();
    case NOP_FALLBACK_INITIALIZATION:
        return NOP_FALLBACK_FACTORY;
    case FAILED_INITIALIZATION:
        throw new IllegalStateException(UNSUCCESSFUL_INIT_MSG);
    case ONGOING_INITIALIZATION:
        // support re-entrant behavior.
        // See also http://jira.qos.ch/browse/SLF4J-97
        return TEMP_FACTORY;
    }
    throw new IllegalStateException("Unreachable code");
}
```

跟进去看performInitialization()

```
private final static void performInitialization() {
    bind();
    if (INITIALIZATION_STATE == SUCCESSFUL_INITIALIZATION) {
        versionSanityCheck();
    }
}
```

重点看bind()方法

```java
private final static void bind() {
    try {
        Set<URL> staticLoggerBinderPathSet = findPossibleStaticLoggerBinderPathSet();
        reportMultipleBindingAmbiguity(staticLoggerBinderPathSet);
        // the next line does the binding
        StaticLoggerBinder.getSingleton();
        INITIALIZATION_STATE = SUCCESSFUL_INITIALIZATION;
        reportActualBinding(staticLoggerBinderPathSet);
        fixSubstitutedLoggers();
    } catch (NoClassDefFoundError ncde) {
        String msg = ncde.getMessage();
        if (messageContainsOrgSlf4jImplStaticLoggerBinder(msg)) {
            INITIALIZATION_STATE = NOP_FALLBACK_INITIALIZATION;
            Util.report("Failed to load class \"org.slf4j.impl.StaticLoggerBinder\".");
            Util.report("Defaulting to no-operation (NOP) logger implementation");
            Util.report("See " + NO_STATICLOGGERBINDER_URL + " for further details.");
        } else {
            failedBinding(ncde);
            throw ncde;
        }
    } catch (java.lang.NoSuchMethodError nsme) {
        String msg = nsme.getMessage();
        if (msg != null && msg.contains("org.slf4j.impl.StaticLoggerBinder.getSingleton()")) {
            INITIALIZATION_STATE = FAILED_INITIALIZATION;
            Util.report("slf4j-api 1.6.x (or later) is incompatible with this binding.");
            Util.report("Your binding is version 1.5.5 or earlier.");
            Util.report("Upgrade your binding to version 1.6.x.");
        }
        throw nsme;
    } catch (Exception e) {
        failedBinding(e);
        throw new IllegalStateException("Unexpected initialization failure", e);
    }
}
```



先看看findPossibleStaticLoggerBinderPathSet（）

```java
private static Set<URL> findPossibleStaticLoggerBinderPathSet() {
    // use Set instead of list in order to deal with bug #138
    // LinkedHashSet appropriate here because it preserves insertion order during iteration
    Set<URL> staticLoggerBinderPathSet = new LinkedHashSet<URL>();
    try {
        ClassLoader loggerFactoryClassLoader = LoggerFactory.class.getClassLoader();
        Enumeration<URL> paths;
        if (loggerFactoryClassLoader == null) {
            paths = ClassLoader.getSystemResources(STATIC_LOGGER_BINDER_PATH);
        } else {
            paths = loggerFactoryClassLoader.getResources(STATIC_LOGGER_BINDER_PATH);
        }
        while (paths.hasMoreElements()) {
            URL path = paths.nextElement();
            staticLoggerBinderPathSet.add(path);
        }
    } catch (IOException ioe) {
        Util.report("Error getting resources from path", ioe);
    }
    return staticLoggerBinderPathSet;
}
```

findPossibleStaticLoggerBinderPathSet()中在没有传入其他classLoader的情况下会默认去查找classpath下的org/slf4j/impl/StaticLoggerBinder.class，项目中如果有log4j,slf4j,logback，那么paths会有三个值。

```java
private static Set<URL> findPossibleStaticLoggerBinderPathSet() {
    // use Set instead of list in order to deal with bug #138
    // LinkedHashSet appropriate here because it preserves insertion order during iteration
    Set<URL> staticLoggerBinderPathSet = new LinkedHashSet<URL>();
    try {
        ClassLoader loggerFactoryClassLoader = LoggerFactory.class.getClassLoader();
        Enumeration<URL> paths;
        if (loggerFactoryClassLoader == null) {
            paths = ClassLoader.getSystemResources(STATIC_LOGGER_BINDER_PATH);
        } else {
            paths = loggerFactoryClassLoader.getResources(STATIC_LOGGER_BINDER_PATH);
        }
        while (paths.hasMoreElements()) {
            URL path = paths.nextElement();
            staticLoggerBinderPathSet.add(path);
        }
    } catch (IOException ioe) {
        Util.report("Error getting resources from path", ioe);
    }
    return staticLoggerBinderPathSet;
}
```

再看 StaticLoggerBinder.getSingleton();编译器会选择一个StaticLoggerBinder.class来完成绑定.再通过StaticLoggerBinder.getSingleton().getLoggerFactory()来获取具体的日志实例。