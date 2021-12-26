---
title: JAVA多线程操作
date: 2019-07-04 15:42:00
categories:
  - JAVA
---

# java 多线程操作
重要类：juc
## Thread 类
`run()`方法和`start()`方法
<!-- more -->
所有的多线程都必须通过start()方法实现
## 虚拟机与底层算法的交互
class在jvm中运行，jvm通过调度不同操作系统的底层函数与操作系统交互。不同的函数借助系统控制算法与底层交互的访问

## 基于Runnable接口实现多线程
Java程序存在单一继承的限制，所以也会通过`java.lang.runnable`接口调度实现多线程

由于接口都使用了函数式接口定义，所以也可以直接利用Lambda表达式调用

优先使用Runnable接口实现，可以避免单继承的局限，更重要的是方便修改增加新功能

实际上Runnable是Thread类的父类，实现Thread也跑不开实现Runnable

## 自定义业务处理类和Thread类的关系
多线程设计之中使用了代理设计模式的结果，用户自定义的线程主体只是负责项目核心工程的实现，其他全交给Thread类处理。

在进行Thread启动多线程的时候调度是`start()`方法，而后找到的是`run()`方法，但是通过Thread类构造方法传递一个Runnable接口对象时，那么该接口对象将被Thread类中的target属性所保存，在`start()`方法在执行的时候调用的是Thread被真实业务处理所覆写的`run()`方法。

多线程开发的本质是在于多个线程可以进行同一资源的抢占，Thread描述的是线程，Runnable描述的是资源。

虽然说作为Runnable子类的Thread也可以进行资源的描述，但是实际之中不会这么做。

## 使用Callable实现多线程
从最传统的开发讲如果要进行多线程的实现依靠的肯定是Runnable，但是Runnable有一个缺点，就是不能明确线程执行完成的情况。因此`java.util.concurrent.Callable`接口实现线程，可以获取一个返回值。

~~~java
@FunctionalInterface
public interface Callable<V>{
	public V call() throws Exception;
}
~~~

可以发现Callable定义的时候可以设置一个泛型，这个泛型就是回传的数据类型，避免类型向下转换的安全风险。

~~~java
import java.util.concurrent.Callable;
import java.util.concurrent.FutureTask;

class MyThread implements Callable<String> {//实现自Callable<-FutureTask<V>类\Future<V>接口、两者都是又RunnableFuture引出的
    @Override
    public String call() throws Exception {
        for (int x=0;x< 10;x++){
            System.out.println("*******线程执行   x="+x);
        }
        return "线程执行完毕";
    }
}

public class CallableDemo {
    public static void main(String[] args) throws Exception {
        FutureTask<String> task = new FutureTask<>(new MyThread());
        new Thread(task).start();
        System.out.println("[线程返回数据]"+task.get());
    }
}

~~~
如上，实现了使用Callable启动线程，并利用FutureTask的`get()`方法得到了线程的返回状态

**总结：Runnable与Callable的区别**

* Runnable是在JDK1.0时提出的多线程实现接口，Callable是在JDK1.5之后提出的
* java.lang.Runnable接口之中只有一个`run()`方法，并且没有返回值
* java.util.Callable接口中提供有`call()`方法，提供使用Futuretask中的`get()`接收一个返回值
* 无论哪种实现，最终都要落实回Thread的`start()`和`run()`


## 线程运行状态
对于多线程的开发而言，编写程序的过程中总是按照：定义线程主体类，而后通过Thread类进行线程调用；但是并不意味着调用了`start()`方法线程就已经开始运行了，因为整体的线程处理总有自己的一套运行状态。

* 任何一个线程的对象都应该使用Thread类进行封装，所以线程的启动使用的是`start()`，但是启动的时候若干线程都将进入到一个就绪状态，不是立即开始执行。
* 进入到就绪状态后，就需要进行资源的调度，当某一个线程调度成功之后则进入到运行状态：`run()`方法，但是不可能一直持续执行下去，中间需要产生一些暂停的状态，例如：某个线程执行一段时间后就要让出资源，进入到阻塞状态，随后重新回归到就绪状态。
* 当`run()`方法执行完毕后，实际上该线程的主要任务也就结束了，那么此时就可以直接进入到停止状态。

# 多线程的主要操作方法
多线程的主要操作方法都在Thread类中定义了。
## 线程的命名与取得
多线程的运行状态是不确定的，那么程序的开发中为了可以获取到一些需要操作的线程，就要依靠线程的名字进行操作。
所以线程的名字很重要。

~~~java
public Thread(Runnable target, String name);//构造方法
public final void setName(String name);//设置名字
public final void getName();//获取名字
~~~

举例如下

~~~java
class myThread implements Runnable{
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName());
    }
}
public class threadDemo2 {
    public static void main(String[] args) {
        myThread mt = new myThread();
        new Thread(mt,"线程A").start();
        new Thread(mt,"线程B").start();
        new Thread(mt,"线程C").start();
    }
}

~~~

每当使用java执行任务，就是启动了一个jvm进程，每一个jvm进程都有一堆线程。在任何的开发中，主线程能创造若干字线程，创建子线程的目的是将复杂逻辑交予子线程处理。

## 线程的休眠
如果希望某个线程可以暂缓执行一次，那么就可以使用休眠来处理

~~~java
public static void sleep(long millis) throws InterruptedException;//按照毫秒休眠
public static void sleep(long millis,int nanos) throws InterruptedException;//精确到纳秒
~~~
`InterruptedException`是`Exception`的子类，休眠抛出的异常必须进行处理，但是休眠时间结束，会自动继续执行。

示例代码如下：

~~~java
class myThread implements Runnable{
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName());
    }
}
public class threadDemo2 {
    public static void main(String[] args) {
        new Thread(()->{
            for(int x = 0;x<10;x++){
                System.out.println(Thread.currentThread().getName()+"x="+x);
                try {
                    Thread.sleep(1000);//sleep()方法需在异常抛出类中调用
                }catch (InterruptedException e){
                    e.printStackTrace();
                }
            }
        },"当前休眠线程对象").start();
    }
}

~~~

**当有多个线程对象在休眠时，仍然是有顺序的。并不是一起运行一起停止。**

实现如下，利用Runnable接口类实现：

~~~java
public class threadDemo2 {
    public static void main(String[] args) throws Exception {
            Runnable run = () -> {
                for (int x = 0; x < 10; x++) {
                    System.out.println(Thread.currentThread().getName() + "、x=" + x);
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            };
            for (int num =0;num<5;num++){//一起休眠，一同启动
                new Thread(run,"线程对象-"+num).start();
            }
    }
}

~~~

## 线程中断
之前发现在休眠时有一个**中断异常**，实际上就证明线程的休眠是可以打断的，这个操作是由**别的线程**操作完成的。

在Thread类中也有提供这种中断执行的处理方法：

~~~java
public boolean isInterrupted();//判断线程是否被中断
public void interrupt();//中断线程执行
~~~

下面用一个线程被打断后暴怒的情况作为例子：

~~~java
import java.util.SimpleTimeZone;

public class threadDemo2 {
    public static void main(String[] args) throws Exception{
        Thread thread = new Thread(() -> {
            System.out.println("****** 疯狂之后就需要休息来补充体力");
            try {
                Thread.sleep(10000);
                System.out.println("****** 睡饱了才能继续出门祸害人");
            } catch (InterruptedException e) {
                System.out.println("尼玛的不让人睡觉，三天之内撒了你");
            }

        });
        thread.start();//开始睡觉
        if(!thread.isInterrupted()){//该线程中断了吗？（有没有睡着啊？？）
            System.out.println("我就要打扰你的睡眠");
            thread.interrupt();//中断执行
        }
    }
}
~~~

**所有的正在执行的线程都是可以被中断的，但中断抛出的异常必须进行处理**

## 线程的强制执行
所谓的线程强制执行是指：**当满足于某些条件之后，某一个线程对象可以一直独占资源，一直到该线程的程序执行结束**

~~~java
public class threadDemo2 {
    public static void main(String[] args) throws Exception {
        Thread thread = new Thread(() -> {
            for (int i = 0; i < 100; i++) {
                System.out.println(Thread.currentThread().getName() + "执行、i = " + i);
            }
        }, "正常的线程");
        thread.start();
        for (int x = 0; x < 100; x++) {
            System.out.println("霸道的main线程  number = " + x);
        }
    }
}
~~~

这个时候两个线程是交替执行的，但是如果说你现在希望主线程独占执行，那么就可以利用强制执行。

在Thread类中有强制执行的方法：

~~~java
public final void join() throws Exception;//强制执行
~~~

在线程强制执行的时候，一定要获取强制执行的线程对象后才能使用`join()`方法进行调用

~~~java
public class threadDemo2 {
    public static void main(String[] args) throws Exception {
        Thread mainThread = Thread.currentThread();//获得主线程
        Thread thread = new Thread(() -> {
            for (int i = 0; i < 100; i++) {
                if (i == 3) {
                    try {
                        mainThread.join();//霸道的线程要先执行
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println(Thread.currentThread().getName() + "执行、i = " + i);
            }
        }, "正常的线程");
        thread.start();
        for (int x = 0; x < 100; x++) {
            System.out.println("霸道的main线程  number = " + x);
        }
    }
}
~~~

## 线程的礼让
指的是先将资源让出去，让别的线程先执行，这个在Thread类中也有这样的方法

~~~java
public class threadDemo2 {
    public static void main(String[] args) throws Exception {
        Thread thread = new Thread(() -> {
            for (int i = 0; i < 100; i++) {
                if(i%3==0){
                    Thread.yield();//线程礼让
                    System.out.println("######正常的线程礼让");
                }
                System.out.println(Thread.currentThread().getName() + "执行、i = " + i);
            }
        }, "正常的线程");
        thread.start();
        for (int x = 0; x < 100; x++) {
            System.out.println("霸道的main线程  number = " + x);
        }
    }
}
~~~

当每次进行礼让的时候，礼让的也只有**当前调用**的资源

## 线程的优先级
从理论上讲，线程的优先级越高，线程越有可能先执行（越有可能抢占的资源）。

在Thread类中有两个主要的方法：

~~~java
public final void setPriority(int newPriority);//设置优先级
public final int getPriority();//获取优先级
~~~

在进行优先级设定的时候通过是int进行定义的，对于此数字的选择在Thread类中有定义三个常量：

~~~java
public static final int MAX_PRIORITY;//最高优先级  10
public static final int NORM_PRIORITY;//中等优先级  5
public static final int MIN_PRIORITY;//最低优先级  1
~~~
但是优先级的设定对线程执行的影响并不是绝对的

~~~java
public class threadDemo2 {
    public static void main(String[] args) throws Exception {
        Runnable run = ()->{
            for(int x =0;x<10;x++){
                System.out.println(Thread.currentThread().getName()+"执行");
            }
        };
        Thread threadA = new Thread(run,"线程对象A");
        Thread threadB = new Thread(run,"线程对象B");
        Thread threadC = new Thread(run,"线程对象C");
        threadA.setPriority(Thread.MIN_PRIORITY);
        threadB.setPriority(Thread.MAX_PRIORITY);//有可能先执行，但不是绝对性的先执行
        threadA.start();
        threadB.start();
        threadC.start();
    }
}
~~~
> 主方法（主线程）和自己创建的线程都是中等优先级

## 线程的同步与死锁处理

在多线程处理时，利用Runnable描述多个线程操作的资源，而Thread描述每一个线程对象，当多个线程访问同一资源的时候如果处理不当就会出现错误。

**范例：实现卖票操作**

~~~java
class MThread implements Runnable {
    private int ticket = 10;
    @Override
    public void run() {
        while (true) {
            if (this.ticket > 0) {
                try {
                    Thread.sleep(100);//模拟网络延迟
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + "卖票，ticket = " + this.ticket--);
            } else {
                System.out.println("*******票已经抢光*******");
                break;
            }
        }
    }
}

public class threadDemo2 {
    public static void main(String[] args) throws Exception {
        MThread mt = new MThread();
        new Thread(mt, "票贩子1").start();
        new Thread(mt, "票贩子2").start();
        new Thread(mt, "票贩子3").start();
    }
}
~~~

此时的程序创建三个线程对象，但是卖票的时候有问题，会卖成负数。

### 线程同步问题的解决关键：锁
指的是当某一个线程执行操作的时候，其他线程在外面等待；

* 如果要想在程序中实现锁功能，就可以使用`synchronized`关键字进行实现。一般进行同步对象的处理的时候，可以采用当前对象this进行同步。**但是程序整体的执行性能会下降**

 ~~~java
 synchronized(this){
  //同步代码块
 }
 ~~~


* 利用同步方法进行解决：只需在方法定义上使用`synchronized`关键字进行解决


 ~~~java
 class MThread implements Runnable {
    private int ticket = 10;

    public synchronized boolean sale() {
        if (this.ticket > 0) {
            try {
                Thread.sleep(100);//模拟网络延迟
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + "卖票，ticket = " +  this.ticket--);
            return true;
        } else {
            System.out.println("*******票已经抢光*******");
            return false;
        }
    }
    @Override
    public void run() {
        while (this.sale()) {
        }
    }
 }
 public class threadDemo2 {
    public static void main(String[] args) throws Exception {
        MThread mt = new MThread();
        new Thread(mt, "票贩子1").start();
        new Thread(mt, "票贩子2").start();
        new Thread(mt, "票贩子3").start();
    }
 }
 ~~~
 
 在学习Java类库时会发现，系统中很多同步处理用的都是这样的同步方法。
 
### 线程的死锁？
所谓的死锁是指若干的线程彼此相互等待的状态。若干个线程访问同一资源的时候一定要进行同步处理，而死锁就是因为同步引起的。
 