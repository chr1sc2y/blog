---
title: "Java notes"
date: 2023-01-31T15:10:14+08:00
draft: true
categories: ["Java"]
---

# Java

## Concept

**Blob**: Binary Large Object

**RuntimeException**: extends `Exception`, but does not need to be caught.

**DAO**: Data Access Object.

**Bean**: class implements `Serializable`, with getter, setter, no-arg constructor.

**MBean**: standard Bean, register to `MBeanServer`, rendered by `HtmlAdapater`.

**JMX**: Java Management Extensions: MBean -> MBeanServer -> HtmlAdapater.

**annotation**: `@xxx` before member functions / variables.

**functional interface**: SAM, single abstract method interface.

## Concurrent

1. `Executors` and `ThreadPoolExecutor` are both under `java.util.concurrent`.

```java
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadPoolExecutor;
```

2. `ThreadPoolExecutor` is the underlying implementation.

```java
public ThreadPoolExecutor(
    int corePoolSize,
    int maximumPoolSize,
    long keepAliveTime,
    TimeUnit unit,
    BlockingQueue<Runnable> workQueue
) {
    ...
}
```

3. `Executor` contains some encapsulations.

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(
        nThreads,
        nThreads,
        0L,
        TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<Runnable>()
    );
}
```

4. Do not use `Executors` to create thread pool directly.

- `FixedThreadPool` and `SingleThreadPool`: `BlockingQueue` size is `Integer.MAX_VALUE`, which may cause OOM.
- `CachedThreadPool` and `ScheduledThreadPool`: `maximumPoolSize` size is `Integer.MAX_VALUE`, which may cause OOM.

5. Example:

```java
Executor ex = Executors.newFixedThreadPool(1); // do not use
Executor ex = new ThreadPoolExecutor(
    5,
    5,
    0,
    TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<>(50)
);
```

Functional interfaces:

```java
Consumer<T> void accept(T t);
BiConsumer<T, U> void accept(T t, U u);
Function<T, R> R apply(T t);
BiFunction<T, U, R> R apply(T t, U u);
Predicate<T> boolean test(T t);
BiPredicate<T, U> boolean test(T t, U u);
Supplier<T> T get();
```

## Thread

### Callable

Since Java 1.5.

```java
// return value
// throws Exception
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

### Runnable

Since Java 1.0.

```java
@FunctionalInterface
public interface Runnable {
    /**
     * When an object implementing interface Runnable is used
     * to create a thread, starting the thread causes the object's
     * run method to be called in that separately executing thread.
     */
    public abstract void run();
}
```

## Exception

- `Error` -> `Throwable` -> `Exception`

```java
public class Throwable implements Serializable
public class Exception extends Throwable
public class Error extends Throwable
```

### NoClassDefFoundError vs NoClassDefFoundException

`NoClassDefFoundException`: JVM cannot load class at run-time using Reflection.

`NoClassDefFoundError`: JVM cannot find class during compiling.

## Dependency Injection

- Dagger: [https://zhuanlan.zhihu.com/p/24454466](https://zhuanlan.zhihu.com/p/24454466)
- Guice
