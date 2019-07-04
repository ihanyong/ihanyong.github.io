
# 


AppClassLoader
ExtClassLoader
bootstrap

双亲委托模式。 
一般Class.getClassLoader() 得到的是一个AppClassLoader， 其parent 是 ExtClassLoader。  ExtClassLoader 的parent 是null。 




##  loadClass 方法详解
默认加载类的顺序是：

- 调用findLoadedClass(String) 方法校验class是否已经被加载
- 调用parent class loader 的 loadClass 方法， 如果parents是null， 则调用findBootstrapClassOrNull方法从bootstrap classloader 加载类
- 如果上面都没有找到类， 则调用findClass方法查找。 


If the class was found using the above steps, and the resolve flag is true, this method will then invoke the resolveClass(Class) method on the resulting Class object.
Subclasses of ClassLoader are encouraged to override findClass(String), rather than this method.
Unless overridden, this method synchronizes on the result of getClassLoadingLock method during the entire class loading process.


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


