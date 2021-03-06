[TOC]

### 原子变量与CAS

之前的 Java 内存模型中提到，一些**看起来是原子**的操作其实不是滴。比如

```java
count++;
```

这一步自增操作在内存中会存在好几个步骤。针对 cnt++ 这类**复合**操作，可以使用并发包中的原子操作类。原子操作类是通过循环 **CAS** 的方式来保证其原子性的。

Atomic 翻译成中文是原子的意思。在化学上，我们知道原子是构成一般物质的最小单位，在化学反应中是不可分割的。在我们这里 Atomic 是指一个操作是不可中断的。即使是在多个线程一起执行的时候，一个操作一旦开始，就不会被其他线程干扰。

所以，所谓原子类说简单点就是具有原子/原子操作特征的类。


并发包 `java.util.concurrent` 的原子类都存放在`java.util.concurrent.atomic`下,如下图所示。

<img src="https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-6/JUC原子类概览.png" alt="JUC原子类概览" style="zoom:80%;" />



#### 原子变量的基本概念

##### 1. **悲观的解决方案（阻塞同步）**

在并发环境下，如果不做任何同步处理，就会有线程安全问题。最直接的处理方式就是**加锁**。我们先看一段 **synchronized** 关键字保证变量原子性的代码：

```java
public class Counter {
    private int count;

    public synchronized void addCount(){
        this.count++;
    }
}
```

简单的 count++ 操作，线程对象首先需要获取到 Counter 类实例的**对象锁**，然后完成自增操作，最后释放对象锁。整个过程中，无论是获取锁还是释放锁都是相当**消耗成本**的，一旦不能获取到锁，还需要**阻塞**当前线程等等。

使用独占锁机制来解决，是一种**悲观的**并发策略，抱着一副“总有刁民想害朕”的态势，每次操作数据的时候都认为别的线程会参与竞争修改，所以直接加锁。同一刻只能有一个线程持有锁，那其他线程就会阻塞。线程的挂起恢复会带来很大的性能开销，尽管 JVM 对于非竞争性的锁的获取和释放做了很多优化，但是一旦有多个线程竞争锁，频繁的阻塞唤醒，还是会有很大的性能开销的。所以，使用 synchronized 或其他重量级锁来处理显然不够合理。

##### 2. **乐观的解决方案（非阻塞同步）**

对于这种情况，我们可以将 count 变量声明成**原子变量**，那么对于 count 的自增操作都可以以**原子的**方式进行，就不存在脏数据的读取了。

乐观的解决方案，顾名思义，就是很大度乐观，每次操作数据的时候，**都认为别的线程不会参与竞争修改**，也不加锁。如果操作成功了那最好；如果失败了，比如中途确有别的线程进入并修改了数据（依赖于冲突检测），也不会阻塞，可以采取一些补偿机制，一般的**策略就是反复重试**。很显然，这种思想相比简单粗暴利用锁来保证同步要合理的多。

**原子变量**最主要的一个特点就是所有的操作都是**原子**的，synchronized 关键字也可以做到对变量的原子操作。只是 synchronized 的成本相对较高，需要获取锁对象，释放锁对象，如果不能获取到锁，还需要阻塞在阻塞队列上进行等待。而如果单单只是为了解决对**变量的原子操作**，建议使用原子变量。

Java 给我们提供了以下几种**原子类型**：

- **AtomicInteger** 和 **AtomicIntegerArray**：基于 **Integer** 类型
- **AtomicBoolean**：基于 Boolean 类型
- **AtomicLong** 和 **AtomicLongArray**：基于 Long 类型
- **AtomicReference** 和 **AtomicReferenceArray**：基于**引用**类型

以下将主要介绍 AtomicInteger 和 AtomicReference 两种类型，AtomicBoolean  和 AtomicLong 的使用和内部实现原理几乎和 AtomicInteger 一样。

原子变量保证了该**变量的**所有操作都是**原子**的，不会因为多线程的同时访问而导致脏数据的读取问题。

##### 3. 原子变量与 synchronized 比较

从思维模式上看，原子变量代表一种==**乐观的非阻塞式**==思维，它假定更新冲突比较少，假定没有别人会和我同时操作某个变量，于是在实际修改变量的值的之前不会锁定该变量，但是修改变量的时候是使用 CAS 进行的，一旦发现冲突，**继续尝试**直到成功修改该变量。

而 synchronized 关键字则是一种**悲观的阻塞式**思维，它假定先更新很可能冲突，认为所有人都会和自己同时来操作某个变量，于是在将要操作该变量之前会**加锁**来锁定该变量，得不到锁的时候进入**等待队列**，因此会有**线程切换**等开销，进而继续操作该变量。



#### CAS

随着硬件指令集的发展，我们可以使用基于**冲突检测的乐观并发策略**：先进行操作，如果没有其它线程争用共享数据，那操作就成功了，否则采取补偿措施（不断地重试，直到成功为止）。这种乐观的并发策略的许多实现都不需要将线程阻塞，因此这种同步操作称为非阻塞同步。

乐观锁需要操作和冲突检测这两个步骤具备原子性，这里就不能再使用互斥同步来保证了，只能靠硬件来完成。硬件支持的原子性操作最典型的是：**比较并交换（Compare-and-Swap，CAS）**。CAS 指令需要有 3 个操作数，分别是内存地址 V、旧的预期值 A 和新值 B。当执行操作时，只有当 V 的值等于 A，才将 V 的值更新为 B。



#### AtomicInteger

##### 1. AtomicInteger 的基本使用

​     首先看它的两个构造函数：

```java
private volatile int value;
// 构造方法一
public AtomicInteger(int initialValue) {
    value = initialValue;
}
// 构造方法二
public AtomicInteger() {}
```

可以看到，我们在通过构造函数构造 AtomicInteger 原子变量的时候，如果指定一个 **int 的参数**，那么该原子变量的值就会被**赋值**，否则就是默认的**数值 0**。

也有获取和设置这个 value 值的方法：

```java
// 非原子操作 用得少
public final int get()
public final void set(int newValue) 
```

当然，这两个方法**并不是原子**的，所以一般也**很少**使用，而以下的这些基于**原子操作的方法**则相对使用的频繁。之所以称为原子变量，是因为它包含一些以**原子方式实现组合操作**的方法。这些方法**都依赖**于下面讲的 ==CAS== 方法。

```java
// 基于原子操作，获取当前原子变量中的值并为其设置新值
public final int getAndSet(int newValue)
// 基于原子操作，比较当前的value是否等于expect，如果是设置为update并返回true，否则返回false
public final boolean compareAndSet(int expect, int update)
// 基于原子操作，获取当前的value值并自增一
public final int getAndIncrement()
// 基于原子操作，获取当前的value值并自减一
public final int getAndDecrement()
// 基于原子操作，获取当前的value值并为value加上delta
public final int getAndAdd(int delta)
// 此外还有一些反向的方法，比如：先自增在获取值的等等
```

下面我们实现一个**计数器**的例子，之前我们使用 synchronized 实现过，现在我们使用原子变量再次实现该问题。

```java
// 自定义一个线程类
public class MyThread extends Thread {
	// 定义一个静态原子变量
    public static AtomicInteger value = new AtomicInteger();

    @Override
    public void run(){
        try {
            Thread.sleep((long) ((Math.random())*100));
            // 原子自增
            value.incrementAndGet();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
// main函数中启动100条线程并让他们启动
public static void main(String[] args) throws InterruptedException {
    Thread[] threads = new Thread[100];
    for (int i = 0; i < 100; i++){
        threads[i] = new MyThread();
        threads[i].start();
    }

    for (int j = 0;j < 100; j++){
        threads[j].join();
    }

    System.out.println("value:" + MyThread.value);
}
```

多次运行会得到相同的结果：

```
value: 100
```

##### 2. AtomicInteger源码解析

###### ① Java8之前

AtomicInteger 的实现原理有点像我们的**包装类**，内部主要操作的是 **value 字段**，这个字段**保存**就是原子变量的数值。value 字段定义如下：

```java
private volatile int value;
```

首先 value 字段被 volatile 修饰，即**不存在内存可见性问题**。由于其内部实现原子操作的代码几乎类似，我们主要学习下 **incrementAndGet** 方法的实现。

```java
public final int incrementAndGet() {
    for (;;) {	// 无限循环
        // 获取当前值
        int current = get();
        // 计算期望的next值
        int next = current + 1;
        // 进行比较
        if (compareAndSet(current, next)) {
            // 只有设置成果才返回
            return next;
        }
    }
}
```

方法体是一个死循环，current 获取到当前原子变量中的值，由于 value 被修饰 volatile，所以不存在内存可见性问题，数据一定是**最新**的。然后 current 加一后赋值给 next，调用 CAS 原子操作判断 value 是否被别的线程修改过，如果还是原来的值，那么将 next 的值赋值给 value 并返回 next，否则重新获取当前 value的值，再次进行判断，**直到操作完成**。

incrementAndGet 方法的一个很核心的思想是，在加一之前先去看看 value 的值是多少，**真正加**的时候再去看一下，如果发现变了，不操作数据，否则为 value 加一。

调用了 compareAndSet 方法。

```java
// ※ CAS方法
public final boolean compareAndSet(int expect, int update) {
     return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

==**compareAndSet** 方法==又被称为 **==CAS==**（**比较并设置**），该方法调用 **unsafe** 的一个 **compareAndSwapInt** 方法，这个方法是 **native**，我们看不到源码，但是我们需要知道该方法完成的一个目标：比较当前原子变量的值**是否等于 expect**，如果是则将其修改为 **update** 并返回 **true**，否则直接返回 false。即**如果与期望值相等才更新，否则不更新**。当然，这个操作本身就是原子的，较为底层的实现。

###### ② Java8

但是在 JDK8 以后，做了一些**优化**，但是最后还是调用的 **compareAndSwapInt** 方法。但基本思想还是没变。

内部依然 有 value 变量保存数值，构造方法传入数值。

```java
private volatile int value;

public AtomicInteger(int initialValue) {
    value = initialValue;
}
```

Java8 中 **incrementAndGet**() 方法。

```java
private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();

public final int incrementAndGet() {
    // U就是UnSafe类的静态实例
    return U.getAndAddInt(this, VALUE, 1) + 1;
}
```

Java8 之后很多操作都放到 Unsafe 类中了。如下：

```java
// UnSafe中的getAndAddInt
@HotSpotIntrinsicCandidate
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!weakCompareAndSetInt(o, offset, v, v + delta));
    return v;
}

// UnSafe中的compareAndSetInt
@HotSpotIntrinsicCandidate
public final native boolean compareAndSetInt(Object o, long offset, int expected; int x);
```

AtomicInteger 类主要利用 **CAS (compare and swap) + volatile 和 native 方法来保证原子操作**，从而避免 synchronized 的高开销，执行效率大为提升。

CAS的原理是拿期望的值和原本的一个值作比较，如果相同则更新成新的值。UnSafe 类的 objectFieldOffset() 方法是一个本地方法，这个方法是用来拿到“原来的值”的内存地址，返回值是 valueOffset。另外 value 是一个volatile变量，在内存中可见，因此 JVM 可以保证任何时刻任何线程总能拿到该变量的最新值。

#### AtomicReference

对于一些**自定义类或者字符串**等这些==**引用类型**==，Java 并发包也提供了原子变量的接口支持。AtomicReference 内部使用**泛型**来实现的。

```java
// 泛型保存引用类型
private volatile V value;

public AtomicReference(V initialValue) {
    value = initialValue;
}

public AtomicReference() {}
```

有关其他的一些原子方法如下：

```java
// 获取并设置value的值为newvalue
public final V getAndSet(V newValue) {
    return (V)unsafe.getAndSetObject(this, valueOffset, newValue);
}
```

AtomicReference 中**少了**一些自增自减的操作，但是对于 value 的修改依然是原子的。



#### 使用FieldUpdater操作非原子变量的字段属性

**FieldUpdater** 允许我们**不必**将字段设置为**原子变量**，利用**反射**直接以**原子方式操作字段**。例如：

```java
// 定义一个计数器
public class Counter {
    // 定义一个普通非原子变量
    private volatile  int count;

    public int getCount() {
        return count;
    }

    public void addCount(){
        // 传入类信息与变量信息
        AtomicIntegerFieldUpdater<Counter> updater  = 
            AtomicIntegerFieldUpdater.newUpdater(Counter.class, "count");
        updater.getAndIncrement(this);
    }
}
```

然后我们创建一百个线程随机调用同一个 Counter 对象的 addCount 方法，无论运行多少次，结果都是一百。这种方式实现的原子操作，对于被操作的变量**不需要**被包装成原子变量，但是却可以**直接以原子方式操作它的数值**。



####  ABA问题

普通的原子变量可能也会存在问题。

我们的原子变量都依赖一个核心的方法，那就是 **CAS**。这个方法最核心的思想就是，更改变量值之前先获取该变量当前最新的值，然后在实际更改的时候再次获取该变量的值，如果没有被修改，那么进行更改，否则**循环**上述操作直至更改操作完成。

假如一个线程想要对变量 count 进行修改，实际操作之前获取 count 的值为 A，此时来了一个线程将 count 值修改为 B，又来一个线程获取 count 的值为 B 并将 count 修改为 A，此时第一个线程**全然不知道** count 的值已经被修改两次了，虽然值还是 A，但是实际上数据**已经是脏**的。

这就是典型的 ABA 问题，一个解决办法是，对 count 的每次操作都记录下当前的一个**时间戳**，这样当我们原子操作 count 之前，不仅查看 count 的最新数值，还记录下该 coun t的**时间戳**，在实际操作的时候，只有在 count 的数值和时间戳都没有被更改的情况之下才完成修改操作。

```java
public static void main(String[] args){
    int count = 0;
    int stamp = 1;
    // 引入时间戳防止ABA问题
    AtomicStampedReference reference = new AtomicStampedReference(count, stamp);
    int next = count++;
    reference.compareAndSet(count, next, stamp, stamp);
}
```

**AtomicStampedReference** 的 CAS 方法要求传入**四个参数**，该方法的内部会同时比较 count 和 stamp，只有这两个值都没有发生改变的前提下，CAS 才会修改 count 的值。




