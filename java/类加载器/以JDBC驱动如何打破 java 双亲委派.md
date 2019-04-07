<center><h3>以JDBC驱动如何打破 java 双亲委派</h3></center>

<h5>1. DriverManager如何加载JDBC</h5>

当java虚拟机启动的时候，会先加载jvm所需要的基础类，即JAVA_HOME/lib/*,JAVA_HOME/ext/lib下的基础jar包，将这些基础类class文件加载 到内存中。

看下面代码:

```java
DriverManager.getConnection("");
```

注意：当调用一个类的静态方法的时候，会触发类的static代码的运行：

```java
/**
 * Load the initial JDBC drivers by checking the System property
 * jdbc.properties and then use the {@code ServiceLoader} mechanism
 */
static {
    loadInitialDrivers();
    println("JDBC DriverManager initialized");
}
```

loadInitialDrivers()代码如下：

```java
private static void loadInitialDrivers() {
    String drivers;
    try {
        drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
            public String run() {
                return System.getProperty("jdbc.drivers");
            }
        });
    } catch (Exception ex) {
        drivers = null;
    }
    // If the driver is packaged as a Service Provider, load it.
    // Get all the drivers through the classloader
    // exposed as a java.sql.Driver.class service.
    // ServiceLoader.load() replaces the sun.misc.Providers()

    AccessController.doPrivileged(new PrivilegedAction<Void>() {
        public Void run() {
            //以java spi的方式加载Driver   违反双亲委派机制 第一处
            ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
            Iterator<Driver> driversIterator = loadedDrivers.iterator();

            /* Load these drivers, so that they can be instantiated.
             * It may be the case that the driver class may not be there
             * i.e. there may be a packaged driver with the service class
             * as implementation of java.sql.Driver but the actual class
             * may be missing. In that case a java.util.ServiceConfigurationError
             * will be thrown at runtime by the VM trying to locate
             * and load the service.
             *
             * Adding a try catch block to catch those runtime errors
             * if driver not available in classpath but it's
             * packaged as service and that service is there in classpath.
             */
            //driversIterator.next();会触发类的初始化阶段，因此会执行jdbc Driver的static代码块
            try{
                while(driversIterator.hasNext()) {
                    driversIterator.next();
                }
            } catch(Throwable t) {
            // Do nothing
            }
            return null;
        }
    });

    println("DriverManager.initialize: jdbc.drivers = " + drivers);

    if (drivers == null || drivers.equals("")) {
        return;
    }
    String[] driversList = drivers.split(":");
    println("number of Drivers:" + driversList.length);
    for (String aDriver : driversList) {
        try {
            println("DriverManager.Initialize: loading " + aDriver);
            //违反双亲委派机制  第二处
            Class.forName(aDriver, true,
                    ClassLoader.getSystemClassLoader());
        } catch (Exception ex) {
            println("DriverManager.Initialize: load failed: " + ex);
        }
    }
}
```

注意：上面方法有两个地方都违法了双亲委派机制。

<b>第二处如何违反了双亲委派机制？</b>

```java
Class.forName(aDriver, true, ClassLoader.getSystemClassLoader());
```

   这个aDriver来自于System.getProperty("jdbc.drivers")，而ClassLoader.getSystemClassLoader()是AppClassLoader，DriverManager是启动类加载器BootstrapClassLoader加载的，却要依赖子加载器AppClassLoader去加载，因此此处违反了双亲委派机制。



<b>第一处如何违反了双亲委派机制呢</b>？

```java
 ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
```

先看看java spi如何加载Driver的

```java
public static <S> ServiceLoader<S> load(Class<S> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}
```

Thread.currentThread().getContextClassLoader() 这个获取的可是线程上下文的类加载。

<b>注</b>：线程上下文如何没有设置classloader，默认就是应用程序加载器即AppClassLoader

因此，这里就打破了双亲委派机制，启动类加载器依赖了子类类加载器去加载。



再深入，顺便了解jdbc如何完成加载的吧。

继续看ServiceLoader.load(Driver.class);

```java
private ServiceLoader(Class<S> svc, ClassLoader cl) {
    service = Objects.requireNonNull(svc, "Service interface cannot be null");
    loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
    acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
    reload();
}
```

```java
public void reload() {
    providers.clear();
    lookupIterator = new LazyIterator(service, loader);
}
```

再看看LazyIterator, DriverManager获取的就是LazyIterator

```java
ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
Iterator<Driver> driversIterator = loadedDrivers.iterator();
```

```java
try{
                while(driversIterator.hasNext()) {
                    driversIterator.next();
                }
            }
```

```java
 public S next() {
            if (acc == null) {
                return nextService();
            } else {
                PrivilegedAction<S> action = new PrivilegedAction<S>() {
                    public S run() { return nextService(); }
                };
                return AccessController.doPrivileged(action, acc);
            }
        }

```

```java
private S nextService() {
    if (!hasNextService())
        throw new NoSuchElementException();
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
        //此处会加载类，这个只会触发加载，验证，解析，不会触发类的初始化
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service,
             "Provider " + cn + " not found");
    }
    if (!service.isAssignableFrom(c)) {
        fail(service,
             "Provider " + cn  + " not a subtype");
    }
    try {
        //newInstance会触发类的初始化
        S p = service.cast(c.newInstance());
        providers.put(cn, p);
        return p;
    } catch (Throwable x) {
        fail(service,
             "Provider " + cn + " could not be instantiated",
             x);
    }
    throw new Error();          // This cannot happen
}
```

现在jdbc driver的已经加载好了，并且已经执行类初始化了。

再看看jdbc driver类初始化做了什么事情

```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    public Driver() throws SQLException {
    }

    static {
        try {
            //绑定到DriverManager中了
            DriverManager.registerDriver(new Driver());
        } catch (SQLException var1) {
            throw new RuntimeException("Can't register driver!");
        }
    }
}
```

至此jdbc 驱动如何通过java spi的方式加载的，以及如何破坏双亲委派机制已经讲解完毕！



总结：

1. 类加载器只负责类的加载阶段，其他阶段不归加载器管。

2. 双亲委派机制的目的，比如rt.jar中的java.lang.Object，无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器进行加载，因此Object类在程序中只会是同一个类，而不会出现多个加载器加载同一个Object，造成混乱。

3. class的唯一性是由这个类及加载这个类的加载器来确定的。

4. 这个打破双亲委派机制，不是发生在加载阶段，而是说，基础类DriverManager，是由启动类加载器加载，运行的时候却要依赖子类加载器去加载，这类基础代码总是被用户代码调用的API,却又要调用回用户的代码，这就打破了双亲委派机制。

5. 双亲委派机制模型的实现：

   ```java
   protected Class<?> loadClass(String name, boolean resolve)
       throws ClassNotFoundException
   {
       synchronized (getClassLoadingLock(name)) {
           // First, check if the class has already been loaded
           Class<?> c = findLoadedClass(name);
           if (c == null) {
               long t0 = System.nanoTime();
               try {
                   if (parent != null) {
                       c = parent.loadClass(name, false);
                   } else {
                       c = findBootstrapClassOrNull(name);
                   }
               } catch (ClassNotFoundException e) {
                   // ClassNotFoundException thrown if class not found
                   // from the non-null parent class loader
               }
   
               if (c == null) {
                   // If still not found, then invoke findClass in order
                   // to find the class.
                   long t1 = System.nanoTime();
                   //自定是类加载器需要自己去实现这个方法
                   c = findClass(name);
   
                   // this is the defining class loader; record the stats
                   sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                   sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                   sun.misc.PerfCounter.getFindClasses().increment();
               }
           }
           if (resolve) {
               resolveClass(c);
           }
           return c;
       }
   }
   ```

