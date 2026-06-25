---
标题: "【Java杂项】synchronized 锁的到底是谁？this、Class 和自定义 lock 怎么选"
创建时间: 2026-06-17
更新时间: 2026-06-17
已发布平台: CSDN
关联知识:
  - "[[021_第二十篇：多线程基础|Java 多线程基础]]"
  - "[[025_T01_volatile 为什么不能保证 count++ 原子性|volatile 为什么不能保证 count++ 原子性]]"
  - "`synchronized`"
  - "`Lock`"
  - "线程安全"
  - "锁对象"
  - "临界区"
tags:
  - 知识库/杂项知识点
  - Java/基础
  - Java/多线程
  - Java/并发
  - 概念/synchronized
  - 概念/锁对象
  - 概念/线程安全
---

> [!info] 知识摘要
> **总结**
> - `synchronized` 的互斥效果取决于“多个线程是否竞争同一个锁对象”，不是取决于代码长得像不像、方法名是不是一样。
> - 普通同步方法使用当前实例 `this` 作为锁；静态同步方法使用当前类的 `Class` 对象作为锁；同步代码块使用括号里表达式得到的对象作为锁。
> - 更精确地说，`synchronized` 是以某个对象作为锁身份，JVM 会围绕这个对象关联/获取 monitor；不要把它理解成对象内部有一个 Java 字段叫 `monitor`。
> - 锁选择的核心问题是：要保护的共享状态属于谁。实例状态用实例锁或私有实例锁，类级共享状态用类锁或静态私有锁，内部临界区优先用 `private final Object lock`。
> - `synchronized` 保证同一把锁下的互斥和可见性，但不解决死锁、不承诺公平顺序，也不会天然提升吞吐量。
>
> **相关知识点**
> - [[021_第二十篇：多线程基础|Java 多线程基础]]、线程安全、临界区、互斥、可见性、对象监视器、`synchronized`、`Lock`

| 写法 | 锁身份 | 适合保护 |
| --- | --- | --- |
| `synchronized void m()` | 当前实例 `this` | 这个对象自己的实例字段 |
| `static synchronized void m()` | 当前类的 `Class` 对象 | 类级共享状态、静态字段 |
| `synchronized (this)` | 当前实例 `this` | 需要和其他实例同步方法共用一把锁时 |
| `synchronized (Xxx.class)` | `Xxx` 的 `Class` 对象 | 需要全类共享的一把锁时 |
| `synchronized (lock)` | 括号里的对象 | 明确设计出来的一把私有锁 |

---

## 一、先看现象：同一个方法不一定是同一把锁

很多人第一次用 `synchronized`，会把它理解成“给方法加了全局排队”。但普通同步方法并不是按方法名加锁，而是按调用它的对象加锁。

```java
class Counter {
    private int count = 0;

    public synchronized void increase() {
        count++;
        System.out.println(Thread.currentThread().getName() + " -> " + count);
    }
}
```

如果两个线程操作的是同一个对象：

```java
Counter counter = new Counter();

new Thread(counter::increase, "A").start();
new Thread(counter::increase, "B").start();
```

线程 A 和线程 B 竞争的是 `counter` 这同一个对象锁，所以会互斥。

但如果两个线程操作的是两个不同对象：

```java
Counter c1 = new Counter();
Counter c2 = new Counter();

new Thread(c1::increase, "A").start();
new Thread(c2::increase, "B").start();
```

线程 A 锁的是 `c1`，线程 B 锁的是 `c2`。两把锁互不相干，所以两个线程可以同时进入 `increase()`。

关键不是“这个方法有没有 `synchronized`”，而是：

```text
所有访问同一份共享数据的线程，是否竞争同一把锁。
```

---

## 二、精确一点：锁对象、monitor 和对象本身的关系

入门时说“`synchronized` 锁的是对象”没有问题，因为从 Java 代码层面看，确实是某个对象决定了锁身份：

```java
synchronized (lock) {
    // 临界区
}
```

这里的 `lock` 表达式会求出一个对象引用，JVM 以这个对象作为同步入口。多个线程只有拿同一个对象做同步入口，才会互斥。

但如果说得更精确一点：

```text
synchronized 不是把对象本身变成一段互斥代码，
也不是说对象内部有一个 Java 字段叫 monitor。
它是以某个对象作为锁身份，JVM 围绕这个对象关联并获取 monitor。
```

对象头中的 Mark Word 会记录与锁相关的状态；在锁膨胀等场景下，JVM 会关联到更完整的 monitor 结构。初学阶段不需要背对象头格式，只要避免两个误解：

| 误解 | 更准确的理解 |
| --- | --- |
| 对象本身就是锁的全部实现 | 对象提供锁身份，JVM 负责关联和管理 monitor |
| 每个对象里都有一个 Java 字段叫 `monitor` | monitor 不是普通 Java 字段，不能通过对象属性访问 |
| 同一个类的方法天然共用一把锁 | 普通同步方法默认锁各自的 `this` |

所以，本文后面说“锁 `this`”“锁 `Xxx.class`”“锁 `lock`”，都是从代码可读性的角度说“用哪个对象作为锁身份”。

---

## 三、先用决策树选锁

写 `synchronized` 前，不要先想语法，先问要保护的共享状态属于谁。

```text
要保护的共享状态是谁？
├─ 只属于某一个实例对象
│  ├─ 可以接受外部也拿这个对象当锁吗？
│  │  ├─ 可以：用 synchronized 实例方法或 synchronized(this)
│  │  └─ 不想暴露：用 private final Object lock
│  └─ 注意：不同实例默认不是同一把锁
├─ 属于整个类，或者是 static 字段
│  ├─ 需要简单写法：用 static synchronized 方法
│  └─ 想隐藏锁对象：用 private static final Object LOCK
├─ 多个对象需要协同保护同一份资源
│  └─ 明确传入或共享同一个 lock 对象
└─ 需要 tryLock、超时、公平锁或多个条件队列
   └─ 考虑 ReentrantLock
```

这棵树背后的原则很简单：

```text
共享数据在哪里，锁的范围就要覆盖到哪里。
锁太小，保护不住；锁太大，竞争变重。
```

后面的所有写法，都可以放回这棵树里判断。

---

## 四、三种 synchronized 写法分别锁谁

### 1. 普通同步方法：锁当前实例

普通同步方法：

```java
class Account {
    public synchronized void withdraw() {
        // 扣款逻辑
    }
}
```

等价于：

```java
class Account {
    public void withdraw() {
        synchronized (this) {
            // 扣款逻辑
        }
    }
}
```

它适合保护这个对象自己的实例字段。比如一个 `Account` 对象有自己的余额，一个 `Cart` 对象有自己的商品列表。

但同一个类的不同实例不会天然互斥：

```java
Account a1 = new Account();
Account a2 = new Account();
```

`a1.withdraw()` 锁的是 `a1`，`a2.withdraw()` 锁的是 `a2`。

### 2. 静态同步方法：锁 Class 对象

静态同步方法：

```java
class IdGenerator {
    public static synchronized int nextId() {
        return 1;
    }
}
```

等价于：

```java
class IdGenerator {
    public static int nextId() {
        synchronized (IdGenerator.class) {
            return 1;
        }
    }
}
```

静态方法不属于某个实例，所以它锁的是 `IdGenerator.class` 这个类对象。它适合保护静态字段、全局计数器、类级缓存这类“全类共享”的状态。

### 3. 同步代码块：锁括号里的对象

同步代码块最灵活：

```java
synchronized (lock) {
    // 临界区
}
```

这里锁的不是变量名 `lock`，而是变量当前指向的对象。只要两个线程进入同步块时，括号里求出来的是同一个对象，它们就会互斥。

这也是为什么锁对象通常要稳定：

```java
class BadCounter {
    private Object lock = new Object();
    private int count = 0;

    public void increase() {
        synchronized (lock) {
            count++;
        }
    }

    public void changeLock() {
        lock = new Object();
    }
}
```

如果 `lock` 被换成新对象，后续线程就会去竞争新锁。旧锁和新锁互不相干，互斥关系会被破坏。

更稳的写法是：

```java
class SafeCounter {
    private final Object lock = new Object();
    private int count = 0;

    public void increase() {
        synchronized (lock) {
            count++;
        }
    }
}
```

`final` 不是让对象“更能锁”，而是防止锁引用被换掉。

---

## 五、实例锁、类锁和私有锁怎么选

最常见的选择，其实就是这三类。

| 要保护的状态 | 推荐锁 | 说明 |
| --- | --- | --- |
| 实例字段，只属于当前对象 | `private final Object lock` 或 `this` | 不希望暴露锁时，用私有锁 |
| 静态字段，全类共享 | `private static final Object LOCK` 或 `Xxx.class` | 不要只用实例锁保护静态状态 |
| 多个方法共享同一段实例状态 | 同一个实例锁 | 普通同步方法之间默认共用 `this` |
| 多份独立状态可以并发处理 | 不同私有锁 | 减小锁粒度，减少无关竞争 |

一个典型错误是用实例锁保护静态变量：

```java
class WrongCounter {
    private static int count = 0;

    public synchronized void increase() {
        count++;
    }
}
```

`count` 是全类共享的，但 `increase()` 锁的是每个实例自己的 `this`。如果业务里创建了多个 `WrongCounter` 对象，多线程仍然可能同时修改同一个静态变量。

更匹配的写法是静态同步方法：

```java
class Counter {
    private static int count = 0;

    public static synchronized void increase() {
        count++;
    }
}
```

或者使用静态私有锁：

```java
class Counter {
    private static final Object LOCK = new Object();
    private static int count = 0;

    public static void increase() {
        synchronized (LOCK) {
            count++;
        }
    }
}
```

这两种写法都把锁范围提升到了类级别，能覆盖所有实例。

---

## 六、为什么工程里更偏向 private final lock

直接锁 `this` 很方便，但 `this` 会暴露给外部代码：

```java
OrderService service = new OrderService();

synchronized (service) {
    // 外部代码也锁住了 service 这个对象
}
```

如果 `OrderService` 内部也用普通同步方法或 `synchronized (this)`，外部代码就可能意外拖住内部逻辑。更麻烦的是，外部调用者未必知道你内部也在锁 `this`，锁关系会变得隐蔽。

所以，内部临界区通常更推荐：

```java
class OrderService {
    private final Object lock = new Object();
    private int stock = 100;

    public boolean reduceStock() {
        synchronized (lock) {
            if (stock <= 0) {
                return false;
            }
            stock--;
            return true;
        }
    }
}
```

这把锁只在类内部可见，语义更单一。

同时，下面这些对象不适合作为锁：

| 不推荐锁的对象 | 原因 |
| --- | --- |
| 可重新赋值的字段 | 引用一变，锁就变了 |
| 字符串常量 | 字符串常量池可能让无关代码共享同一对象 |
| 包装类对象 | 可能有缓存，也容易被当成值对象使用 |
| 可能为 `null` 的对象 | `synchronized (null)` 会直接抛 `NullPointerException` |
| 外部传入或第三方对象 | 你不知道别的代码是否也拿它当锁 |

锁对象最好满足三个条件：

```text
稳定、私有、语义单一。
```

---

## 七、实现侧补充：为什么 synchronized 能自动释放锁

这一节只作为理解边界，不影响前面的锁选择。

从 JVM 实现看，同步方法和同步代码块的形式不同：

| 写法 | JVM 层面的直观理解 |
| --- | --- |
| 同步方法 | 方法元数据带有同步标记，调用时先获取对应锁 |
| 同步代码块 | 字节码里出现 `monitorenter` / `monitorexit` |

可以把同步代码块理解成：

```text
monitorenter：尝试获取与锁对象关联的 monitor
monitorexit：退出同步块，释放 monitor
```

如果同步块里抛出异常，JVM 也会保证退出路径释放锁。这就是 `synchronized` 比手写 `Lock` 更省心的地方：不用自己写 `unlock()`。

但自动释放只发生在代码块正常结束或异常退出时。如果线程在同步块里死循环、长时间 IO、等待远程服务，它仍然会一直占着锁。

---

## 八、synchronized 保证什么，不保证什么

`synchronized` 解决的是“同一把锁保护下的临界区访问问题”。

它主要保证两件事：

| 能力 | 含义 |
| --- | --- |
| 互斥 | 同一时刻只有一个线程能持有同一把锁并进入对应同步区域 |
| 可见性 | 一个线程释放锁前的修改，对后续获取同一把锁的线程可见 |

所以，`count++` 放进同一把 `synchronized` 锁里可以安全，是因为读、加、写这几步被放进了互斥临界区，并且退出锁后修改结果对后续拿锁线程可见。

但它不保证这些事：

| 不保证 | 说明 |
| --- | --- |
| 不自动避免死锁 | 两个线程以相反顺序拿多把锁，仍然可能死锁 |
| 不承诺公平顺序 | 等得久的线程不一定先拿到锁 |
| 不提升吞吐量 | 锁会让临界区串行化，粒度过大反而会拖慢 |
| 不修复错误的锁选择 | 不是同一把锁，写再多 `synchronized` 也保护不住 |
| 不替你设计线程协作 | 等待/通知、队列、线程池仍要按场景选择工具 |

因此，使用 `synchronized` 时要同时问两件事：

```text
这段代码是不是必须互斥？
所有访问同一份共享数据的路径，是不是用了同一把锁？
```

---

## 九、和 Lock 怎么选

`synchronized` 和 `Lock` 都能保护临界区，但它们解决问题的姿势不同。

`synchronized` 简单、内置、自动释放：

```java
synchronized (lock) {
    // 临界区
}
```

`Lock` 更灵活，但必须手动释放：

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

class Service {
    private final Lock lock = new ReentrantLock();

    public void work() {
        lock.lock();
        try {
            // 临界区
        } finally {
            lock.unlock();
        }
    }
}
```

可以这样选：

| 场景 | 推荐 |
| --- | --- |
| 只需要保护一小段共享状态 | 优先 `synchronized` |
| 希望代码简单，异常时自动释放 | 优先 `synchronized` |
| 需要尝试加锁、超时放弃 | `ReentrantLock` |
| 需要公平锁 | `ReentrantLock(true)` |
| 需要多个条件队列 | `Lock` + `Condition` |

不要因为 `Lock` 看起来更“高级”就替换所有 `synchronized`。普通临界区里，`synchronized` 往往更清楚，也更不容易忘记释放锁。

---

## 十、面试里怎么答

如果面试问：`synchronized` 锁的到底是谁？

可以这样答：

```text
synchronized 的互斥效果取决于锁对象。普通同步方法锁的是 this，静态同步方法锁的是当前类的 Class 对象，同步代码块锁的是括号里表达式得到的对象。

更精确地说，Java 代码里是用某个对象作为锁身份，JVM 会围绕这个对象关联并获取 monitor；monitor 不是对象里的普通 Java 字段。只有多个线程竞争同一个锁对象时，互斥才会生效。

所以两个线程调用同一个实例的 synchronized 方法会互斥；如果分别调用两个不同实例的方法，就不是同一把锁。静态同步方法和 synchronized(ClassName.class) 是类锁，所有实例共享。实例锁和类锁不是同一把锁。

工程上一般推荐用 private final Object lock = new Object() 作为内部锁，避免锁 this、字符串常量、包装类或可能变化的字段。synchronized 保证互斥和可见性，但不解决死锁、不承诺公平，也不会提升吞吐量。
```

如果继续追问 `synchronized` 和 `Lock` 的区别，可以补充：

```text
synchronized 是 Java 关键字，进入和退出由 JVM 管理，代码块结束或异常退出时会自动释放锁；Lock 是 J.U.C 提供的接口，通常用 ReentrantLock，需要手动 lock 和 unlock，必须放在 finally 里释放。

Lock 提供 tryLock、超时等待、公平锁、Condition 等更灵活的能力；synchronized 更简单，适合普通临界区。二者都要关注锁粒度和是否锁住同一个共享状态。
```

---

## 十一、总结

`synchronized` 的核心不是“写在哪里”，而是“用哪个对象作为锁身份”。

| 误区 | 正确认知 |
| --- | --- |
| 方法加了 `synchronized` 就全局互斥 | 普通同步方法只锁当前实例 |
| 两个对象调用同一个同步方法会互斥 | 不会，它们锁的是不同的 `this` |
| 静态同步方法和实例同步方法会互斥 | 不会，类锁和实例锁不是同一把锁 |
| `synchronized(lock)` 锁的是变量名 | 锁的是变量当前指向的对象 |
| 对象里有一个普通字段叫 `monitor` | monitor 由 JVM 围绕对象锁身份关联和管理 |
| 锁对象随便选都行 | 锁对象要稳定、私有、语义单一 |
| `synchronized` 能解决所有并发问题 | 它保证互斥和可见性，不负责死锁、公平性和吞吐量 |

最后用这句话收束：

```text
线程安全不是“哪里都加 synchronized”，而是“同一份共享数据的所有访问路径，都竞争同一把合适的锁”。
```

