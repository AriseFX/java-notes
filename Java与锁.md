Java中的锁大致分为语法糖级别的`synchronized关键字`与JUC包下的`AQS`

从一段代码开始:

```java
public class JavaLock {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(() -> {
            try {
                Thread.sleep(5000);
                System.out.println("new thread end");
            } catch (InterruptedException ignored) {
            }
        });
        thread.start();
        thread.join();
        System.out.println("main end");
    }
}
```

熟悉Java多线程的一定知道程序执行的结果为：

```
(阻塞5s)
new thread end
main end
```

本文对原理进行解释，涉及到 Java标准库的源码、synchronized相关的JVM源码、POSIX线程库

#### **Thread类中的`join()`方法**

```java
    //代码经过了精简
public class Thread {

    public final synchronized void join(long millis) {
        if (millis == 0) {
            while (isAlive()) {
                wait(0);
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
}
```
在当前测试用例中，join的作用是：`让main线程进入阻塞状态，直到new Thread终结`。`isAlive()`是Thread类中方法，判断的是new Thread是否活跃，
而`wait()`则是Java中主动让当前线程（main线程）陷入阻塞状态的一种途径。

join()方法在Java语言层面的实现非常简单，需要深入的点主要有三个：

**1.为什么wait需要synchronized？**
     
首先synchronized需要一个锁对象，这里是new Thread对象，



**2.为什么wait要放在while循环里?**
    
**3.wait()方法是如何让当前线程进入阻塞状态的?**

   