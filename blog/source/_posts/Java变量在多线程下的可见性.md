---
title: Java变量在多线程下的可见性
date: 2022-02-25 23:58:43
tags: java happens-before
---
在多线程的情况下，Java一个变量被更改了是否能够立即被另外的线程看到？即使是单个线程，判断一个变量不为空，那么此时这个变量真的为空吗？这就涉及到了变量可见性、指令重排的问题。更深层次的是指令重排。

<!--more-->
## 一、 问题
有时候我们会通过全局变量共享数据，有两个问题要搞清楚：
1、`可见性问题`，线程A可能一直不结束，为什么呢？
2、`指令重排`，线程A结束的时候，为何有时候 shareVal输出的是 0，而不是5？
**示例代码1**
```java
public class Application {
    private static boolean needRun = true;
    private static int shareVal = 0;
    public static void main(String[] args) throws InterruptedException {
       // thread A
        new Thread(() - > {
            while (needRun) { // 1
                Thread.yield();
                System.out.println("A run...");
            }
            System.out.println("A end shareVal is " + shareVal); // 2
        }).start();
        // 睡眠一会，先让上面的线程跑起来
        Thread.sleep(1000);
        // thread B
        new Thread(() - > {
            shareVal = 5;  // 3
            needRun = false; // 4
            System.out.println("B change needRun to false ok...");
        }).start();

        Thread.sleep(1000 * 60);
    }
}
```
3、`又一个指令重排`，如果下面第二行不加 volatile，假设可见性没问题的情况下。如果第8行`if (singleton == null)`不成立的时候，返回的单例实例，为什么有时候是不完整的，明明写的代码(3)在(2)后面，为何singleton != null的时候，仍返回不完整的实例？
例如单例模式如下：
**示例代码2**
```java
 1 public class Singleton5 {
 2         private volatile static Singleton singleton;
 3  
 4         private SingletonDemo5() {
 5         }
 6  
 7         public static Singleton newInstance() {
 8             if (singleton == null) {
 9                 synchronized (Singleton.class) {
10                     if (singleton == null) {
                            // 可以看做是三行代码
                            // allocator memeort into instance (1)
                            // init instance (2)
                            // singleton = instance (3)
11                         singleton = new Singleton();
12                     }
13                 }
14             }
15             return singleton;
16         }
17     }
```

## 二、相关知识
### 1、Java 内存模型与可见性

1、线程对共享变量的所有操作都必须在自己的工作内存中进行,不能直接从主内存中读写。
2、线程间的变量值传递需要主内存来完成。
- 执行线程 A 的处理器把变量 V 缓存到寄存器中。
- 执行线程 A 的处理器把变量 V 缓存到自己的缓存中，但还没有同步刷新到主内存中去。
- 执行线程 B 的处理器的缓存中有变量 V 的旧值。

不过在现代可共享内存的多处理器体系结构中每个处理器都有自己的缓存，并周期性的与主内存协调一致。不会一直看不到的（是否能够最终一致，待求证）

### 2、指令重排
为了提高执行效率，CPU和编译器可能对指令进行重排，使得实际执行的顺序与代码中的顺序不同。

### 3、happens-before原则
happends-before原则不局限于语言，是指令执行的通用概念。这个概念就一句话：
前一个操作的结果对后一个操作可见。无论二者是否在同一个线程。 (the first is visible to and ordered before the second)。

happends-before 有传递原则，以下面代码为例：

例如，
- 如果说 1 2 heppends-before 3，
- 3 happends-before 4 (即3操作对4可见)， 那么1 2 同样 happends-before 4。当4读到true的时候，因为只有3能设置为true，所以说明3发生了。1 2 
通过传递性， happends-before 4，所以1 2 此时也可见了。

```java
private int x = 0;
private int y = 1;
private volatile boolean flag = false;

public void writer() {
    x = 42; //1
    y = 50; //2
    flag = true; //3
}

public void reader() {
    if (flag) { //4
        System.out.println("x:" + x); //5
        System.out.println("y:" + y); //6
    }
}
```

## 三、示例分析

### 1、可见性

复制一份上面的**示例代码1**到这，分析一波
```java
public class Application {
    private static boolean needRun = true;
    private static int shareVal = 0;
    public static void main(String[] args) throws InterruptedException {
       // thread A
        new Thread(() - > {
            while (needRun) { // 1
                Thread.yield();
                System.out.println("A run...");
            }
            System.out.println("A end shareVal is " + shareVal); // 2
        }).start();
        // 睡眠一会，先让上面的线程跑起来
        Thread.sleep(1000);
        // thread B
        new Thread(() - > {
            shareVal = 5;  // 3
            needRun = false; // 4
            System.out.println("B change needRun to false ok...");
        }).start();

        Thread.sleep(1000 * 60);
    }
}
```
**问题分析**

由于线程工作的原理，当thread B 更改了`needRun = fasle`之后，trhead A并不会立即看到，所以A不会立即停止，甚至JVM会优化掉while(needRun)为while(true)

**解决**
通过happens-beofre原则可知，只要thread B 写 变量needRun的时候，happens-before线程A读needRun即可。
如果保证，有几个方法
1. 对一个 volatile 域的写, happens-before 于任意`后续`对这个 volatile 域的读
这个实现方法是：即写一个被volatile修饰的变量，发生两个事情：
- 立即将当前线程中缓存的新值刷回贮存
- 刷写回主内存的操作会使其他CPU缓存的该共享变量内存地址的数据无效。

所以，给needRun变量加上 volatile修饰符即可。
2. 一个锁的解锁 happens-before 于随后对这个锁的加锁。
所以可以每次对needRun读写的时候加`synchronized`，当needRun = fasle发生时，涉及到了加锁解锁，一定happends-before下次一对needRun的加锁，即while循环里下一次读needRun

3. 多说一句， 如果线程 A 执行操作 ThreadB.start() (启动线程B), 那么 A 线程的 ThreadB.start() 操作 happens-before 于线程 B 中的任意操作，也就是说，主线程 A 启动子线程 B 后，子线程 B 能看到主线程在启动子线程 B 前的操作。

### 2、指令重排

还是示例代码1，因为 3 和 4 之间没有依赖关系，所以编译、执行的时候都有可能重排，先执行`needRun = false`，让 A 结束（假设没有可见性问题了），后执行 shareVal = 5；结果就是A打印出了 shareVal = 0

再看示例代码2，如果去掉volatile
```java
 1 public class Singleton5 {
 2         private volatile static Singleton singleton;
 3  
 4         private SingletonDemo5() {
 5         }
 6  
 7         public static Singleton newInstance() {
 8             if (singleton == null) {
 9                 synchronized (Singleton.class) {
10                     if (singleton == null) {
                            // 可以看做是三行代码
                            // allocator memeort into instance (1)
                            // init instance (2)
                            // singleton = instance (3)
11                         singleton = new Singleton();
12                     }
13                 }
14             }
15             return singleton;
16         }
17     }
```
**问题**
(3)与(2)之间没有依赖关系，所以可能先执行(3)，使得 singleton != null，此时(2)还没有执行，所以返回了一个不完整的实例。

**解决**

1、用volatile修饰singleton，volatile语义增强后，会禁止将它`写`之前的操作重排的后面，所以禁止(2)重排到(3)后面。

## 四、结论

happens-before能保证可见性。
volatile这个关键字有两个大的作用：
- 确定了对被修饰变量本身读写的happens-before关系。
- 增强的语义，添加了内存屏障，影响了指令重排：
1.  volatile 写之前的操作不会被重排到volatile之后
2.  volatile 读之后的操作不会被重排序到 volatile 读之前

所以通过共享变量通信很是麻烦，golang倡导通过通信共享数据，而不是通过共享数据进行通信。
Do not communicate by sharing memory; instead, share memory by communicating.

下面的文章详细解释了happends-before
[Java可见性与指令重排](https://juejin.im/post/5d80251d518825491b72419c)
[golang内存模型与可见性](https://golang.org/ref/mem)
