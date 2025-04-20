---
title: Interesting details in the Hikari CP source code
date: 2025-04-20 15:12:45
tags: Java
---
# 藏在HikariCP中的有趣细节 Chapter One
> HikariCP是Java生态中的一个流行的数据库连接池。它也是Spring框架中默认的数据库连接池。
> 对应的源码可以在[这里找到](https://github.com/brettwooldridge/HikariCP)

## Hotspot is fucking amazing.
> 下图是 詹姆斯・高斯林（James Gosling）在Java One上的演讲截图。
![hotspot.png](Interesting-details-in-the-Hikari-CP-source-code/hotspot.png)

它的源码结构简单，找到连接池对象后我们查看它的构造方法。
```java
/**
    * Construct a HikariPool with the specified configuration.
    *
    * @param config a HikariConfig instance
    */
   public HikariPool(final HikariConfig config)
   {
      super(config);

      this.connectionBag = new ConcurrentBag<>(this);
      // isAllowPoolSuspension是创建连接池时的可选参数。
      // 这个选项我从未使用过，也似乎从未见其他人使用过。
       // 从后面的阅读我们可以反推出这个选项是表示是否允许在获取数据库连接进行等待。
      this.suspendResumeLock = config.isAllowPoolSuspension() ? new SuspendResumeLock() : SuspendResumeLock.FAUX_LOCK;
      // Ignore other...
   }
```
根据不同的选项，`suspendResumeLock` 被赋予了不同的对象实例。

### SuspendResumeLock 源码查看
```java
public class SuspendResumeLock {
    // 这个语法等同于实例化一个该类的子类。这个子类重写了SuspendResumeLock的所有方法且方法内部均为空。
   public static final SuspendResumeLock FAUX_LOCK = new SuspendResumeLock(false) {
      @Override
      public void acquire() {}

      @Override
      public void release() {}

      @Override
      public void suspend() {}

      @Override
      public void resume() {}
   };

   private static final int MAX_PERMITS = 10000;
   // 你可以简单的将这个类理解为对这个信号量的操作的包装。
   private final Semaphore acquisitionSemaphore;
    /**
     * Default constructor
     */
    public SuspendResumeLock()
    {
        this(true);
    }

    private SuspendResumeLock(final boolean createSemaphore)
    {
        acquisitionSemaphore = (createSemaphore ? new Semaphore(MAX_PERMITS, true) : null);
    }
   // Ignore Four methods of SuspendResumeLock.
}
```
全局检索可以知道，整个HikariCP中只有在获取数据库链接的时候会用到这个类。
```java
public Connection getConnection(final long hardTimeout) throws SQLException {
    suspendResumeLock.acquire();
    // Ignore other... 
    try{
        // Ignore 
    }catch (Exception e){
        // Ignore
    }
    finally {
        suspendResumeLock.release();
    }
}
```
根据它的构造方法可以得知：当你手动设置了`isAllowPoolSuspension`的时候，获取数据库连接将不会使用到信号量。

但是为什么要这么写呢？直接通过一个全局的布尔值应该也可以实现和上述等同的逻辑。
> 不过我个人还是喜欢其原来的写法。上面的代码看起来紧凑好看~

```java
public Connection getConnection(final long hardTimeout) throws SQLException {
    if (isAllowPoolSuspension)
        suspendResumeLock.acquire();
    // Ignore other... 
    try{
        // Ignore 
    }catch (Exception e){
        // Ignore
    }
    finally {
        if (isAllowPoolSuspension)
            suspendResumeLock.release();
    }
}
```
#### JIT优化
其实真要扣性能的话，上述的代码显然不如源码的实现。这是因为JIT的*C2*能将`FAUX_LOCK`移除掉。
注意到`FAUX_LOCK`本身是被`static final`修饰的，这意味着它是一个常量。当C2介入时，`getConnection`的代码会被优化成如下代码：
```java
public Connection getConnection(final long hardTimeout) throws SQLException {
//    suspendResumeLock.acquire();  // 由于其方法是空，C2会将其移除掉
    // Ignore other... 
    try{
        // Ignore 
    }catch (Exception e){
        // Ignore
    }
    finally { 
//        suspendResumeLock.release();  // C2内联优化
    }
}
```
#### 如果用于判断的布尔值是被final修饰的呢？
也许聪明的你会试图为刚刚的假想做打一个补丁：

```java
final boolean isAllowPoolSuspension = false;
public HikariPool(final HikariConfig config) {
    super(config);
    isAllowPoolSuspension = config.isAllowPoolSuspension();
}
public Connection getConnection(final long hardTimeout) throws SQLException {
    if (isAllowPoolSuspension)
        suspendResumeLock.acquire();
    // Ignore other... 
    try{
        // Ignore 
    }catch (Exception e){
        // Ignore
    }
    finally {
        if (isAllowPoolSuspension)
            suspendResumeLock.release();
    }
}
```
`final`在Java中的语义也等同于不可变。照上述的说法，C2也可以将对应的代码抹除掉。

可惜的是，`final`在Java中并不意味着真正的final。

具体可以阅读以下几个链接：
> [红帽的JVM答疑](https://shipilev.net/jvm/anatomy-quarks/15-just-in-time-constants/)
> 
> [JEP: Prepare to Make Final Mean Final](https://openjdk.org/jeps/8349536)
