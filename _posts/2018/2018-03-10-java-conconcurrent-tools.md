---
layout:       post
title:        "Java并发工具"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Java
---

java并发工具包来自jdk 1.5，它使得Java的并发编程变得更加容易。

下面介绍一些常见的API，更多请查阅官方文档。

## 数据结构和辅助类：

### BlockingQueue

阻塞队列是一种特殊的队列。这种队列是有限的。

阻塞队列适用于消费者生产者模型：

- 生产线程可以向阻塞队列插入数据，如果队列已满，那么生产线程会被阻塞。
- 消费线程可以从阻塞队列取出数据，如果队列是空的，那么消费线程被阻塞。

BlockingQueue包含四种处理队列满或者队列空的机制：

- 抛异常：如果队列满试图写或队列空试图读(以下简称"越界")，抛出一个异常
- 返回值：如果越界，返回false或null，正常返回true或对象
- 阻塞：如果越界，阻塞线程
- 超时：设定一个时间，如果越界，等待这段时间。如果时间耗尽队列还是满的或空的则返回false或null。

所以BlockingQueue包含了四组方法，分别对应于上面的每个越界处理机制：

- 插入数据
  - 抛异常：boolean add(E e); 满时抛出IllegalStateException
  - 返回：boolean offer(E e);
  - 阻塞：void put(E e) throws InteruptedException;
  - 超时：offer(E e, long time, TimeUnit timeunit);
- 取出数据
  - 抛异常：E remove();  空时抛出NoSuchElementException
  - 返回：E poll();  空时返回null
  - 阻塞：E take() throws InterruptedException;
  - 超时：E poll(long time, TimeUnit timeunit);

BlockingQueue是一个接口。它有如下常用实现类：

- ArrayBlockingQueue: 队列底层基于数组实现
- LinkedBlockingQueue: 队列底层基于链表实现
- DelayQueue: 当指定时间到了才能取出元素
- PriorityBlockingQueue: 具有优先级的阻塞队列
- SynchronousQueue: 内部同时只能容纳单个元素的队列

#### ArrayBlockingQueue 和 LinkedBlockingQueue

Array和Linked的用法和链表很像。区别仅在于底层实现上。

下面演示插入数据：

```java

public class BlockingQueueDemo {

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(15);
        for (int i = 0; i < 15; ++i) {
            queue.add(i);
        }

        // when out of bound:
        // queue.add(100);  // throw IllegalStateException
        // System.out.println("return " + queue.offer(100));  // return false
        // queue.put(100);  // blocking...
        System.out.println("return " +
                queue.offer(100, 5, TimeUnit.SECONDS));
    }

}

```

注意比对在越界时四种方法的处理机制。前几种不难理解，留意offer(E e, long time, TimeUnit timeunit)这个方法，在越界时，该方法会让线程阻塞若干时间，如果在时间过了以后，队列还是满的，那么方法会返回false；如果队列有空位，那么方法会插入数据并且返回true。

取出数据的操作也很简单：

```java

public class BlockingQueueDemo {

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(15);

        queue.put(10);

        System.out.println(queue.remove());
        // System.out.println(queue.remove());  // NoSuckElementException
        // System.out.println(queue.take());  // blocking
        // System.out.println(queue.poll());  // null
        System.out.println(queue.poll(5, TimeUnit.SECONDS));
    }

}

```

LinkedBlockingQueue的用法和ArrayBlockingQueue类似。只不过在实例化的时候可以不指定最大长度。这样其在理论上是"无界"的（实际上长度是int的最大值）。

当然，LinkedBlockingQueue也可以手动指定一个最大值，这样用法就和ArrayBlockingQueue一样了。

#### PriorityBlockingQueue

优先级队列指的是保存的元素都有一个优先级。优先级是按照排序序列确定的。对于保存自己定义的类的序列，是按照Comparable接口制定的规则实现的。

对这样的队列进行取出操作默认是先取出优先级别高的元素：

```java

class Student implements Comparable<Student> {

    private String name;
    private int score;

    public Student(String name, int score) {
        this.name = name;
        this.score = score;
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", score=" + score +
                '}';
    }

    @Override
    public int compareTo(Student o) {
        return o.score - this.score;
    }
}

public class BlockingQueueDemo {

    public static void main(String[] args) throws InterruptedException {

        BlockingQueue<Student> queue = new PriorityBlockingQueue<>(5);

        queue.add(new Student("张三", 90));
        queue.add(new Student("李四", 20));
        queue.add(new Student("王五", 100));
        queue.add(new Student("牛逼", 9));

        // 注意取出的顺序
        for (Student student : queue) {
            System.out.println(student);
        }

    }

}

```

观察到取出数据的时候是分数高的Student先取出，和插入的顺序没有关系。

### ConcurrentMap

我们知道Hashtable是线程安全的。但是其中大部分都是通过直接上互斥锁实现的。也就是一个线程对Hashtable操作的时候整个集合都是被锁住的。在高并发情况下效率是难以让人接受的。

ConcurrentMap是加强版的Hashtable。区别在于锁的粒度，也就是ConcurrentMap有着更细粒化的锁。

最好的情况是针对元素上锁(类似数据库的行级锁)，但是实现起来过于复杂。且程序要花大量精力在管理锁上，有点不值得。

ConcurrentMap内部把整个集合拆分成多个子集合。对集合的某个元素进行操作的时候，ConcurrentMap会把该元素所在的子集合锁住，而不是整个集合。也就是所谓的“分段锁”的原则。

ConcurrentMap是一个接口。常用其ConcurrentHashMap实现类。用法和一般的Map一样，这里不再演示。

理论上说，ConcurrentMap的效率最多比Hashtable高了16倍。

### ConcurrentNavigableMap

ConcurrentNavigableMap是一种特殊的Map，特殊之处在于它可以取出指定的子Map。

它有三个很有意思的方法：

- headMap(key): 表示取出从key到头部的元素。按照key的顺序取出。不包括key本身。
- tailMap(key): 表示取出从key到尾部的元素。按照key的顺序取出，包括key本身。
- subMap(key1, key2): 表示取出从key1到key2的元素，按照key的顺序取出，包括key1但是不包括key2。

注意这里key的顺序是按照key排序的顺序，不是插入元素的顺序。

看下面的例子，就能明白这三个方法的使用和区别了：

```java

public class ConcurrentNavigableMapDemo {

    public static void main(String[] args) {
        ConcurrentNavigableMap<String, String> map =
                new ConcurrentSkipListMap<>();
        map.put("b", "2");
        map.put("c", "3");
        map.put("a", "1");
        map.put("d", "4");
        map.put("e", "5");
        map.put("f", "6");

        ConcurrentNavigableMap<String, String> headMap = map.headMap("c");
        System.out.println("headMap = " + headMap);

        ConcurrentNavigableMap<String, String> tailMap = map.tailMap("d");
        System.out.println("tailMap = " + tailMap);

        ConcurrentNavigableMap<String, String> subMap = map.subMap("b", "e");
        System.out.println("subMap = " + subMap);
    }

}

```

输出是：

```console
headMap = {a=1, b=2}
tailMap = {d=4, e=5, f=6}
subMap = {b=2, c=3, d=4}
```

### CountDownLatch

CountDownLatch, 闭锁。是一个并发构造，它一般用于一个或多个线程等待一系列指定操作完成再做某事的场景。

初始化一个CountDownLatch对象需要一个给定的数量。每调用一次countDown()方法，这个数量就会减一。调用await()方法，线程可以阻塞并且等待这个数量减到0。

下面模拟一个简单的场景：在出门前，需要穿上衣服，裤子和鞋。做这三件事情的顺序无所谓（这里不讨论穿好鞋后穿裤子不方便的情况），但是要出门这三件事必须保证是做好的，同时我们假设这三件事情可以同时做（每个线程做一件事情）。那么这样的场景就很适合使用CountDownLatch了：

```java

public class CountDownLatchDemo {

    public static void main(String[] args) throws InterruptedException {

        // 要做3件准备工作
        CountDownLatch count = new CountDownLatch(3);

        // 启动三个线程开始准备工作
        new Thread(new WearClothes(count)).start();
        new Thread(new WearPants(count)).start();
        new Thread(new WearShoes(count)).start();

        // 此处产生阻塞，直到count变为0
        count.await();

        System.out.println("我出门啦！");
    }

}

class WearClothes implements Runnable {

    private CountDownLatch count;
    WearClothes(CountDownLatch count) {
        this.count = count;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(300);
            System.out.println("衣服穿好啦！");
            count.countDown();   // 完成一件事，count减一
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

class WearPants implements Runnable {

    private CountDownLatch count;
    WearPants(CountDownLatch count) {
        this.count = count;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(500);
            System.out.println("裤子穿好啦！");
            count.countDown();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

class WearShoes implements Runnable {

    private CountDownLatch count;
    WearShoes(CountDownLatch count) {
        this.count = count;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(1000);
            System.out.println("鞋子穿好啦！");
            count.countDown();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```

这样，在衣服，裤子，鞋子中有任何一个没有穿好的情况下。main线程是不可能执行到出门那里的。

### CyclicBarrier

CyclicBarrier, 也叫栅栏。其一般适用于多个线程先进行自己的准备工作，在所有线程的准备工作做完前，栅栏允许这些线程卡在某个执行点（想象它们被栅栏拦住）。在所有线程的准备工作做完以后，这些线程才可以从执行点继续往下执行代码。

调用await()方法后，线程会阻塞在调用处，并让CyclicBarrier的计数器减一。直到CyclicBarrier的计数器减到0，阻塞才会被解除。

一个很好理解的场景是百米赛跑。我们知道在赛跑前所有运动员需要做准备活动，在所有运动员准备活动做好之前，运动员是不能够跑的。那我们就可以在起跑点设置一个栅栏，在所有运动员准备好之后才下令，让运动员开始跑步。

为了增加随机性，下面的示例将产生5个运动员，他们的准备时间是随机的（不大于5秒）。在所有人准备好以后，所有运动员开始跑步：

```java

public class CyclicBarrierDemo {

    public static void main(String[] args) {
        CyclicBarrier barrier = new CyclicBarrier(5);  // 假设有5个运动员
        for (int i = 1; i <=5; ++i) {
            Random random = new Random();
            new Thread(new Athlete(barrier, "运动员" + i,
                    random.nextInt(5000))).start();
        }
    }

}

class Athlete implements Runnable {

    private CyclicBarrier barrier;
    private String name;
    private long prepareTime;
    Athlete(CyclicBarrier barrier, String name,
            long prepareTime) {
        this.barrier = barrier;
        this.name = name;
        this.prepareTime = prepareTime;
    }

    @Override
    public void run() {
        System.out.println(name + "准备中...");
        try {
            Thread.sleep(prepareTime);
            System.out.println(name + "准备就绪！");

            // 在其它线程准备好之前，该线程将被阻塞在这里
            barrier.await();

            System.out.println(name + "开始跑步了！");

        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
}

```

我们可以多执行几次这段代码，会发现这5个线程都非常守规矩，在其它线程准备好之前，它们是绝对不会"抢跑"的。

### Exchanger

Exchanger, 交换机。表示一种两个线程之间可以互相交换对象的情况。

交换动作是由exchange()方法实现的。在调用这个方法的时候，线程会发生阻塞，直到有接收到其它线程发过来的对象。

例如我们可以模拟租借影片的情景。假设有一个租赁店和一个顾客，店把影片光盘借给顾客，顾客把钱交给租赁店：

```java

public class ExchangeDemo {

    public static void main(String[] args) {
        Exchanger<String> exchanger = new Exchanger<>();
        new Thread(new Shop(exchanger)).start();
        new Thread(new Customer(exchanger)).start();
    }

}

class Shop implements Runnable {

    private Exchanger<String> exchanger;
    Shop(Exchanger<String> exchanger) {
        this.exchanger = exchanger;
    }

    @Override
    public void run() {
        String CD = "The Avengers";  // 要交换的对象，这里简化为一个字符串
        try {
            String result = exchanger.exchange(CD);
            System.out.println("借出了一张CD, 收获：" + result);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

class Customer implements Runnable {

    private Exchanger<String> exchanger;
    Customer(Exchanger<String> exchanger) {
        this.exchanger = exchanger;
    }

    @Override
    public void run() {
        String money = "100块钱";
        try {
            String result = exchanger.exchange(money);
            System.out.println("交出100块钱，收获：" + result);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```

执行这段代码，我们会发现线程正常地交换了对象。交易是完全公平进行的。

注意在超过两个线程以上交互数据不适合使用Exchanger。

## 线程池：

### ExecutorService

线程池是一个非常重要的工具。它可以大大提高线程的重用性，并且减少线程创建和销毁的开销。

ExecutorService就是一个线程池工具，它是一个接口，维护了下面四种数据：

- 常用线程：也叫核心线程。表示经常要用到的线程，可以指定个数。这些线程会一直保持存在。客户连接上时若有空余的常用线程，则优先使用常用线程。
- 等待阻塞队列：在常用线程被占满后，客户连接上时会检查等待队列有没有满，若没有满则进入等待队列等待其它客户释放线程资源。
- 临时线程：这些线程不会事先创建，但是ExecutorService会维护一个最大临时线程数量。当客户连接上时，若常用线程被占满且等待队列也满了，那么会启动一个临时线程为这个客户服务。在服务结束后若等待队列有数据则继续服务等待队列的客户，如果没有客户了会等待若干时间，若该时间内还是没人连接才销毁临时线程。
- 拒接服务：如果当客户连接上时，常用线程满了，等待队列满了，临时线程也满了，那么会进入拒绝服务的处理。这个处理可以由用户自己定制。

在实际中，需要根据情况分配常用线程、等待队列、临时线程最大数量。

构造一个自定义ExecutorService则使用ThreadPoolExecutor实现类。构造这个实现类需要用到以下参数：

- corePoolSize: 核心线程数量
- maximumPoolSize: 池中的最大线程数，也就是核心线程数+临时线程数
- keepAliveTime: 如果临时线程空闲了，等待多长时间才销毁这个线程
- unit: 上面这个参数的单位
- workQueue: 类型是BlockQueue&lt;Runnable&gt;，表示等待队列。
- handler: 类型是RejectedExecutionHandler接口。接口里面有一个rejectedExecution(Runnable r, ThreadPoolExecutor executor)方法。其表示的就是在等待队列和临时线程都满的情况下执行的操作。

构造完以后，调用submit(Runnable r)方法就可以向线程池申请线程处理任务了。

关闭线程池可以调用shutdown()，注意这个操作不会马上关闭线程池，而是不再接收新的任务了，线程池会等待已经申请的任务全部处理完成之后才关闭。

下面的示例展示了不断向ExecutorService请求线程的情况：（使用了jdk 1.8的lamda表达式，jdk 1.8以下可以用匿名内部类替换）

```java

public class ExecutorServiceDemo {

    public static void main(String[] args) {
        ExecutorService service = new ThreadPoolExecutor(5, 10,
                60, TimeUnit.SECONDS, new ArrayBlockingQueue<>(5),
                (r, executor) -> System.out.println("拒绝服务：线程池已经满了！"));

        for (int i = 0; i < 18; ++i) {  // 18个请求，超出最大线程+等待队列的长度
            service.submit(new TestTask());
        }

        service.shutdown();
    }

}

// 一个一直睡眠的任务，模拟长用户请求
class TestTask implements Runnable {

    @Override
    public void run() {
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```

控制台输出：

```console
拒绝服务：线程池已经满了！
拒绝服务：线程池已经满了！
拒绝服务：线程池已经满了!
```

线程池最多允许10个线程，有5个等待位置。而这段代码申请了18个任务。所以会有3个任务进入拒绝服务。

我们不一定要自己定义线程池，java提供了一些已经指定好参数的线程池供我们使用，调用Executors的一些静态方法可以获取这些线程池：

- newFixedThreadPool(int nThreads)
  - nThreads既是核心线程数又是最大线程数，也就是说这个线程池不允许有临时线程。
  - 等待队列使用LinkedBlockingQueue，并且没有指定最大值。也就是说队列是允许无限长的。（int最大值）
  - 这是一个"小池子大队列"的线程池。适用于减轻服务器压力的场景。
- newCachedThreadPool()
  - 没有核心线程，临时线程无限大（int最大值），临时线程等待时间60s。
  - 等待队列使用SynchronousQueue，也就是只允许一个任务等待。
  - 这是一个"大池子小队列"的线程池。适用于高并发场景。

其余线程池大家可以自行查阅文档和源代码。

### Callable

ExecutorService的submit()方法还可以接受一个Callable实例。

Callable接口和Runnable的区别在于Callable有一个泛型。其包含一个call()方法，原型如下：

- public T call() throws Exception;

这个方法和run()的区别在于它可以返回一个值并且允许将异常直接抛出。

Callable不能通过Thread的构造方法传入并且直接start()启动。它只能通过线程池启动。

通过ExecutorService的submit(Callable c)可以启动一个Callable任务。这个方法会返回一个Future\<T>对象。

Future\<T>有一个get()方法，可以获取Callable的执行结果。

## 锁：

### Lock

锁是保证在并发情况下数据正常的一个重要工具。如果有多个线程同时对一个数据进行读或者写，就有可能出现"脏读"的情况。

在传统的Java并发编程中，可以使用synchronized对操作加上互斥锁使得操作原子化。

Lock是并发工具包提供的锁，是一个接口，其有很多实现类。

Lock的基本操作是：

- lock(): 表示加锁，线程在执行到这里的时候如果锁是空闲的，则持有锁；如果锁已经被其它线程持有了，则休眠。
- unlock()：表示释放锁，线程在执行到此处的时候释放手头的锁。在有异常的情况下建议在finally代码块执行这个方法。

synchronized是由Java虚拟机实现的锁，这是一种公平锁。也就是在线程释放锁以后，会等待其它线程唤醒之后再一起参与抢占锁。

这种锁在高并发情况下吞吐量较低，效率不高。

另一种锁是非公平锁，它允许插队，也就是在线程释放锁之后，可以不等其它线程唤醒完毕就让线程再抢占一次CPU。这样在线程唤醒的过程中可以再执行若干次计算。所以这种锁又叫做可重入锁。

还有一种读写锁。它包括了读锁和写锁。在读取数据的时候，锁是不互斥的，也就是读锁可以由多个线程持有，读操作是完全并发的。而写锁是互斥的，只能由一个线程持有，写操作是原子的。

可重入锁和读写锁分别是ReentrantLock和ReadWriteLock。它们均是Lock的实现子类。

ReadWriteLock可以通过writeLock()方法获取写锁；通过readLock()方法获取读锁。

下面是可重入锁的示例：

```java

public class LockDemo {

    static String name = "卢本伟";
    static String info = "挂逼";

    public static void main(String[] args) {
        Lock lock = new ReentrantLock();
        new Thread(new Changer(lock)).start();
        new Thread(new Reader(lock)).start();
    }

}

class Changer implements Runnable {

    private Lock lock;
    Changer(Lock lock) {
        this.lock = lock;
    }

    @Override
    public void run() {
        for (;;) {
            lock.lock();  // 加一个读锁
            if ("卢本伟".equals(LockDemo.name)) {
                LockDemo.name = "PDD";
                LockDemo.info = "骚猪";
            }
            else {
                LockDemo.name = "卢本伟";
                LockDemo.info = "挂逼";
            }
            lock.unlock();  // 释放锁
        }
    }
}

class Reader implements Runnable {

    private Lock lock;
    Reader(Lock lock) {
        this.lock = lock;
    }

    @Override
    public void run() {
        for (;;) {
            lock.lock();
            System.out.println("name = " + LockDemo.name +
                ", info = " + LockDemo.info);
            lock.unlock();
        }
    }
}

```

以上代码是不会出现脏读的，因为我们进行了合理的上锁和解锁。

读写锁只需要换一下类型然后通过：

- lock.readLock().lock();
- lock.writeLock().lock();

来启动读锁或写锁。通过：

- lock.readLock().unlock();
- lock.writeLock().unlock();

来释放读锁或写锁。

### 原子操作

我们知道，传统的"++"或"--"操作是有两步的，例如"a++"有如下两步(真实情况下并不是这样，这里只是给一个模拟)：

- b = a
- a = b + 1

在并发情况下，这样是非线程安全的。因为在这两个操作之间可能有其它线程对a进行了改动，这样b就是一个"脏数据"了。那么计算的结果将会是错误的。

看以下代码：

```java

public class AtomDemo1 {

    static int a = 0;

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch count = new CountDownLatch(2);
        new Thread(new Adder1(count)).start();
        new Thread(new Adder1(count)).start();

        count.await();
        System.out.println("计算结果：" + a);
    }

}

class Adder1 implements Runnable {

    private CountDownLatch count;
    Adder1(CountDownLatch count) {
        this.count = count;
    }

    @Override
    public void run() {
        for (int i = 1; i <= 1000; ++i) {
            AtomDemo1.a++;
        }
        count.countDown();
    }
}

```

最后输出的结果有机率是小于2000的，甚至很离谱。（笔者就遇到过978的情况）

但是对于++这种简单的操作上锁显得有点没有必要，那么就可以让原来的int变为原子类型。

int的原子类型是AtomicInteger。这个类型可以保证所有操作都是原子的。那么"++"操作就变成了addAndGet(int)和getAndAdd(int)。

这就可以保证类型是完全线程安全的。

那么上述代码就可以改为：

```java

public class AtomDemo1 {

    static AtomicInteger a = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch count = new CountDownLatch(2);
        new Thread(new Adder1(count)).start();
        new Thread(new Adder1(count)).start();

        count.await();
        System.out.println("计算结果：" + a);
    }

}

class Adder1 implements Runnable {

    private CountDownLatch count;
    Adder1(CountDownLatch count) {
        this.count = count;
    }

    @Override
    public void run() {
        for (int i = 1; i <= 1000; ++i) {
            AtomDemo1.a.addAndGet(1);
        }
        count.countDown();
    }
}

```

这样无论如何输出的结果都是2000。

更多原子类型和其操作请参阅API文档。
