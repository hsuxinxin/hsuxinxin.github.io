---
layout: post
category : [并发编程]
tagline: "Supporting tagline"
tags : [java, Synchronized, 多线程]
---
{% include JB/setup %}

在Java中，我们都知道关键字synchronized可以用于实现线程间的互斥，但我们却常常忘记了它还有另外一个作用，那就是确保变量在内存的可见性 - 即当读写两个线程同时访问同一个变量时，synchronized用于确保写线程更新变量后，读线程再访问该 变量时可以读取到该变量最新的值。

比如说下面的例子：

    public class NoVisibility {
        private static boolean ready = false;
        private static int number = 0;

        private static class ReaderThread extends Thread {
            @Override
            public void run() {
                while (!ready) {
                    Thread.yield(); //交出CPU让其它线程工作
                }
                System.out.println(number);
            }
        }

        public static void main(String[] args) {
            new ReaderThread().start();
            number = 42;
            ready = true;
        }
    }

你认为读线程会输出什么？ 42？ 在正常情况下是会输出42. 但是由于重排序问题，读线程还有可能会输出0 或者什么都不输出。

我们知道，编译器在将Java代码编译成字节码的时候可能会对代码进行重排序，而CPU在执行机器指令的时候也可能会对其指令进行重排序，只要重排序不会破坏程序的语义 -

在单一线程中，只要重排序不会影响到程序的执行结果，那么就不能保证其中的操作一定按照程序写定的顺序执行，即使重排序可能会对其它线程产生明显的影响。
这也就是说，语句"ready=true"的执行有可能要优先于语句"number=42"的执行，这种情况下，读线程就有可能会输出number的默认值0.

而在Java内存模型下，重排序问题是会导致这样的内存的可见性问题的。在Java内存模型下，每个线程都有它自己的工作内存（主要是CPU的cache或寄存器），它对变量的操作都在自己的工作内存中进行，而线程之间的通信则是通过主存和线程的工作内存之间的同步来实现的。

比如说，对于上面的例子而言，写线程已经成功的将number更新为42，ready更新为true了，但是很有可能写线程只同步了number到主存中（可能是由于CPU的写缓冲导致），导致后续的读线程读取的ready值一直为false，那么上面的代码就不会输出任何数值。而如果我们使用了synchronized关键字来进行同步，则不会存在这样的问题，

    public class NoVisibility {
        private static boolean ready = false;
        private static int number = 0;
        private static Object lock = new Object();

        private static class ReaderThread extends Thread {
            @Override
            public void run() {
                synchronized (lock) {
                    while (!ready) {
                        Thread.yield();
                    }
                    System.out.println(number);
                }
            }
        }

        public static void main(String[] args) {
            synchronized (lock) {
                new ReaderThread().start();
                number = 42;
                ready = true;
            }
        }
    }
    
这个是因为Java内存模型对synchronized语义做了以下的保证，

![multithread](/img/multithread/1.png)

即当ThreadA释放锁M时，它所写过的变量（比如，x和y，存在它工作内存中的）都会同步到主存中，而当ThreadB在申请同一个锁M时，ThreadB的工作内存会被设置为无效，然后ThreadB会重新从主存中加载它要访问的变量到它的工作内存中（这时x=1，y=1，是ThreadA中修改过的最新的值）。通过这样的方式来实现ThreadA到ThreadB的线程间的通信。

这实际上是JSR133定义的其中一条happen-before规则。JSR133给Java内存模型定义以下一组happen-before规则，

1.单线程规则：同一个线程中的每个操作都happens-before于出现在其后的任何一个操作。

2.对一个监视器的解锁操作happens-before于每一个后续对同一个监视器的加锁操作。

3.对volatile字段的写入操作happens-before于每一个后续的对同一个volatile字段的读操作。

4.Thread.start()的调用操作会happens-before于启动线程里面的操作。

5.一个线程中的所有操作都happens-before于其他线程成功返回在该线程上的join()调用后的所有操作。

6.一个对象构造函数的结束操作happens-before与该对象的finalizer的开始操作。

7.传递性规则：如果A操作happens-before于B操作，而B操作happens-before与C操作，那么A动作happens-before于C操作。

实际上这组happens-before规则定义了操作之间的内存可见性，如果A操作happens-before B操作，那么A操作的执行结果（比如对变量的写入）必定在执行B操作时可见。

为了更加深入的了解这些happens-before规则，我们来看一个例子：

    //线程A，B共同访问的代码
    Object lock = new Object();
    int a=0;
    int b=0;
    int c=0;

    //线程A，调用如下代码
    synchronized(lock){
        a=1; //1
        b=2;  //2
    } //3
    c=3; //4


    //线程B，调用如下代码
    synchronized(lock){  //5
        System.out.println(a);  //6
        System.out.println(b);  //7
        System.out.println(c);  //8
    }
    
我们假设线程A先运行，分别给a,b,c三个变量进行赋值(注：变量a,b的赋值是在同步语句块中进行的），然后线程B再运行，分别读取出这三个变量的值并打印出来。那么线程B打印出来的变量a,b,c的值分别是多少？

根据单线程规则，在A线程的执行中，我们可以得出1操作happens before于2操作，2操作happens before于3操作,3操作happens before于4操作。同理，在B线程的执行中，5操作happens before于6操作，6操作happens before于7操作,7操作happens before于8操作。而根据监视器的解锁和加锁原则，3操作（解锁操作）是happens before 5操作的（加锁操作），再根据传递性 规则我们可以得出，操作1，2是happens before 操作6，7，8的。

则根据happens-before的内存语义，操作1，2的执行结果对于操作6，7，8是可见的，那么线程B里，打印的a，b肯定是1和2. 而对于变量c的操作4，和操作8. 我们并不能根据现有的happens before规则推出操作4 happens before于操作8. 所以在线程B中，访问的到c变量有可能还是0，而不是3.