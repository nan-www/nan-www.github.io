---
title: Functional programming in Java
date: 2024-12-01 07:12:10
tags:
---
# Don't repeat yourself
在公司低版本的RPC框架（Jet）中并未提供相应的语法糖导致每次涉及并发编程时都需要做如下的事
```java
    // 获取泳道、mesh等环境变量
    String env = flowControlContainer.getEnv();
    // 获取当前请求的LogID
    String logId = RequestIDUtil.getCurrentRequestID();
    
    CompletableFuture.runAsync(() ->{
        // 手动设置环境变量
            doSometing();
            });
    
    @Async
    public void doAsync(env, logId){
        // 手动设置环境变量
    }
    
```
没有人喜欢机械式的重复劳动，特别是程序员。另外，正如标题所言这违反了软件工程的一大原则。

# Java中的函数式
如果你写过Go语言的话你的第一直觉会是想把上述的`doSomething()`当成一个**function**传入到线程池中。

虽然Java并不像Go一样有**函数是一等公民**的思想，但是Java有自己的方式来支持**函数式编程**。

在此之前我想以一个例子做引：
```java
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
queryWrapper.eq("name", "老王");

LambdaQueryWrapper<User> lambdaQueryWrapper = new LambdaQueryWrapper<>();
lambdaQueryWrapper.eq(User::getName, "老王");
```
上述是Mybatis-plus的查询示例。可以看到普通的`wrapper`需要手动指定数据库表对应列的名称，`lambdaWrapper`却是传入一个方法来绕开硬编码。
> 你有想过这是如何实现的吗？

# 抽取公共部分
在抽象之前我们先假设上述的`doSomething()`只接受一个入参`T`并返回一个`CompletableFuture`。

由于环境变量和logId都是存储在`ThreadLocal`中的，所以对外暴露的方法并不需要调用者手动传入。

根据上述的假设，封装后对外暴露的方法如下

```java
public <T, R> CompletableFuture<R> supplyAsync(T param, Function<T, R> task){
        // Get log ID and env from current thread
        String logId = getLogId();
        Env env = getEnv();
    return CompletableFuture.supplyAsync(() ->{
        try {
            setLogId(logId);
            setEnv(env);
            return task.apply(param);
        }catch (Exception e){
            // Print error
            return null;
        }finally {
            removeLogId();
            removeEnv();
        }
    });
    }
```

## 如何使用
```java
public class Example {
    private AsyncUtil asyncUtil;

    public Example(AsyncUtil asyncUtil) {
        this.asyncUtil = asyncUtil;
    }

    public Integer print(String s){
        return Integer.valueOf(s);
    }

    public void doAsync(){
        asyncUtil.supplyAsync("11", this::print);
    }
}
```
## Method References in Java
你可能会对`::`这个语法感到迷糊。比如在上述的代码示例中你将`doAsync()`方法改成如下所示
```java
public void doAsync(){
        asyncUtil.supplyAsync("11", Example::print);
    }
```
那么编译时会报错。这是因为方法引用在Java中有一定的约束，让我们来看看Oracle官网给出的规范：

| 类型                                                                                         | 表达式 | 示例 |
|--------------------------------------------------------------------------------------------|-----|----|
| 指向静态方法                                                                                     |  ContainingClass::_staticMethodName_   | MethodReferencesExamples::appendStrings   |
| 指向一个具体的对象的方法                                                                               |   containingObject::instanceMethodName  |  myApp::appendStrings2  |
| Reference to an instance method of an arbitrary object of a particular type（没懂是啥意思所以翻译不了:( | ContainingType::methodName    |  String::compareToIgnoreCase<br/>String::concat  |

以下是附带的代码示例
```java
import java.util.function.BiFunction;

public class MethodReferencesExamples {
    
    public static <T> T mergeThings(T a, T b, BiFunction<T, T, T> merger) {
        return merger.apply(a, b);
    }
    
    public static String appendStrings(String a, String b) {
        return a + b;
    }
    
    public String appendStrings2(String a, String b) {
        return a + b;
    }

    public static void main(String[] args) {
        
        MethodReferencesExamples myApp = new MethodReferencesExamples();

        // Calling the method mergeThings with a lambda expression
        System.out.println(MethodReferencesExamples.
            mergeThings("Hello ", "World!", (a, b) -> a + b));
        
        // Reference to a static method
        System.out.println(MethodReferencesExamples.
            mergeThings("Hello ", "World!", MethodReferencesExamples::appendStrings));

        // Reference to an instance method of a particular object        
        System.out.println(MethodReferencesExamples.
            mergeThings("Hello ", "World!", myApp::appendStrings2));
        
        // Reference to an instance method of an arbitrary object of a
        // particular type
        System.out.println(MethodReferencesExamples.
            mergeThings("Hello ", "World!", String::concat));
    }
}
```
回到上述的问题，你现在应该知道为什么不能将代码中的`this`替换成类的名字了。因为我声明的`print`方法并不是一个静态变量。

使用this关键字即是上述表格中的第二类：**指向一个具体的对象的方法**。
