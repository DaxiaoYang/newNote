# ThreadLocal源码解析

[TOC]

## 线程安全

> 线程安全定义：
>
> class is thread-safe when it continues to behave correctly when accessed
>
> from multiple threads
>
> Correctness means that a class *conforms to its specifification*.



**实现线程安全的方法**

- 加锁：保证修改数据的操作的原子性

- CAS + volatile：也是保证修改数据的操作的原子性

- 使用不可变对象：使得对象在创建之后就不可修改，不存在并发修改问题

- Copy-On-Write

- 使用线程私有数据：线程操作的数据只有线程能访问，其他线程访问不到

  而ThreadLocal则是JDK提供的帮助我们存取线程私有数据的类
  
  Thread Confinement：线程封闭实现方法
  
  - Ad-hoc：volatile 变量作为标志位（保证可见性） 需要保证只有一个线程执行写操作
  - 栈封闭：因为栈是线程独立的
  - ThreadLocal类
  
  





## ThreadLocal的用途

- 实现线程安全，避免数据被并发修改

- 缓存

  如果某个线程一直存活（比如线程池中的核心线程），且需要频繁用到某个数据，则可以将数据存储到线程私有数据中 线程对象对应的内存的区域ThreadLocalMap

- 线程数据避免显式传参(同一线程内方法互相调用时) 保存执行上下文信息

- 作为对象池：对象使用完之后清除信息 放回，再次使用时取出即可



**使用示例**

```java
import java.util.concurrent.atomic.AtomicLong;

/**
 * @ description:利用ThreadLocal生成线程的ID
 * @ author: daxiao
 * @ date: 2021/9/16
 */
public class ThreadId {

  	// 生成进程内唯一的线程ID
    private static final AtomicLong nextId = new AtomicLong(0);

    // 使用静态工厂方法withInitial() 生成一个重写了initialValue方法的ThreadLocal的子类
    private static final ThreadLocal<Long> threadId = ThreadLocal.withInitial(nextId::getAndIncrement);

    // 每个调用该静态方法的线程都获取了唯一的ID 由CAS保证ID生成的原子性
    public static long get() {
        return threadId.get();
    }
}
```





## 源码

**使用流程**

1. 继承`ThreadLocal`类， 在子类中重写 `initialValue()`方法

   为`get()`方法设置初始值，获取子类对象

2. 线程调用调用`threadLocal.get()`获取初始值

3. `threadLocal.set(newValue)` 设置新值

4. 使用完毕后 如果有必要则`threadLocal.remove()` 

   线程池中的线程私有数据如果不想被其他线程看到，用完之后需要remove掉，因为线程会被调用者复用



### 初始化

```java
// ThreadLocal对象只是作为Thread对象中的一个map中的key使用 
// 所以存储不同对象的ThreadLocal设置为静态成员变量即可
private static ThreadLocal<Integer> cache = ThreadLocal.withInitial(() -> 0);

// withInitial是一个静态工厂方法 实际上返回的是一个重写了initialValue方法的子类
static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

  private final Supplier<? extends T> supplier;

  SuppliedThreadLocal(Supplier<? extends T> supplier) {
    // fail fast
    this.supplier = Objects.requireNonNull(supplier);
  }

  @Override
  protected T initialValue() {
    return supplier.get();
  }
}

```



**ThreadLocalMap两个关键点**

- 解决哈希冲突的方式：线性探测

- 使用弱引用作为map中entry的Key以实现垃圾回收 

  stale状态：需要被回收的状态，如果`entry.get() == null` 即Entry中的key ThreadLocal类已经被垃圾回收，被WeakRefence指向的对象，如果只有WeakReference指向它，就会被回收，则清理entry中的value和entry自身，将引用置空，进行垃圾回收（WeakReference不能防止所引用的对象不被垃圾回收）





### 获取 

```java
Integer integer = cache.get();

/*
	实际存储数据的是Thread对象中的ThreadLocal.ThreadLocalMap threadLocals
	ThreadLocal类只是作为一个获取值的一个Key
*/
public T get() {
  // 获取当前线程对象
  Thread t = Thread.currentThread();
  // 获取实际存储线程私有数据的map t.threadLocals;
  ThreadLocalMap map = getMap(t);
  if (map != null) {
    // map中的key为当前的ThreadLocal对象 
    ThreadLocalMap.Entry e = map.getEntry(this);
    if (e != null) {
      @SuppressWarnings("unchecked")
      T result = (T)e.value;
      return result;
    }
  }
  return setInitialValue();
}


private T setInitialValue() {
  // 获取初始化 未重写则为null
  T value = initialValue();
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null)
    map.set(this, value);
  else
    // 延迟初始化 线程中的map
    createMap(t, value);
  return value;
}


void createMap(Thread t, T firstValue) {
  t.threadLocals = new ThreadLocalMap(this, firstValue);
}

ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
  // 创建初始容量为16的Entry数组 容量 保证为2的倍数 用于将取余操作转化为速度较快的bit与操作
  // 如capacity=16 16 - 1 = 15 二进制为 0b1111 任何其他数跟0b1111相与 所得结果就位于[0-15]这个区间中
  table = new Entry[INITIAL_CAPACITY];
  // 哈希值的生成由一个全局的AtomicInteger 间隔一定的距离给出
  int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
  table[i] = new Entry(firstKey, firstValue);
  size = 1;
  // loadfactor是 2/3 如果是16 则在size为10时进行rehash 
  setThreshold(INITIAL_CAPACITY);
}


// 实际执行的在哈希表中的get方法
private Entry getEntry(ThreadLocal<?> key) {
  int i = key.threadLocalHashCode & (table.length - 1);
  // 根据哈希值 获取对应slot中的entry 如果key相同 即threadLocal对象相同 则直接返回
  // 直接比较引用值的方法最快 在HashMap类中也是最先比较引用值再调用object.equals方法
  // fast path 大多数情况下 一个线程的ThreadLocal不会很多 所以冲突不会很多
  Entry e = table[i];
  if (e != null && e.get() == key)
    return e;
  else
    // 否则进行线性探测
    return getEntryAfterMiss(key, i, e);
}


private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
  Entry[] tab = table;
  int len = tab.length;
  // 遇到为null的entry或者是相应的对象
  while (e != null) {
    ThreadLocal<?> k = e.get();
    if (k == key)
      return e;
    // 遇到entry不为null 但是key为null的情况 key为弱引用 当只有这里的弱引用指向threadLocal对象时 在垃圾回收时 不管内存是否够不够用 对象都会被清理掉 
    if (k == null)
      expungeStaleEntry(i);
    else
      //  nextIndex中就是进行环形的线性探测 将数组在逻辑上视为一个环 ((i + 1 < len) ? i + 1 : 0)
      i = nextIndex(i, len);
    e = tab[i];
  }
  return null;
}


/*
	清除当前entry 并且一直nextIndex直到遇到null 
	对所遇到的entry 如果是stale状态: key为null的(说明该ThreadLocal对象引用信息已经被清除 对象已被回收) 直接清空entry
	不为stale状态的进行rehash
*/
private int expungeStaleEntry(int staleSlot) {
  Entry[] tab = table;
  int len = tab.length;
  // expunge entry at staleSlot
  tab[staleSlot].value = null;
  tab[staleSlot] = null;
  size--;
  // Rehash until we encounter null
  Entry e;
  int i;
  for (i = nextIndex(staleSlot, len);
       (e = tab[i]) != null;
       i = nextIndex(i, len)) {
    ThreadLocal<?> k = e.get();
    if (k == null) {
      e.value = null;
      tab[i] = null;
      size--;
    } else {
      // 因为有Entry被清除了 所以可能原来会造成哈希冲突 现在不会了
      int h = k.threadLocalHashCode & (len - 1);
      if (h != i) {
        tab[i] = null;
        // 尽可能将数据放到正确位置上 降低get方法的寻址时间
        while (tab[h] != null)
          h = nextIndex(h, len);
        tab[h] = e;
      }
    }
  }
  return i;
}
```



### 设置

```java
// set和remove方法同样也是操作ThreadLocalMap
public void set(T value) {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null)
    map.set(this, value);
  else
    createMap(t, value);
}

private void set(ThreadLocal<?> key, Object value) {
  Entry[] tab = table;
  int len = tab.length;
  int i = key.threadLocalHashCode & (len-1);

  for (Entry e = tab[i];
       e != null;
       e = tab[i = nextIndex(i, len)]) {
    ThreadLocal<?> k = e.get();

    if (k == key) {
      e.value = value;
      return;
    }
		// 清理stale状态的entry
    if (k == null) {
      replaceStaleEntry(key, value, i);
      return;
    }
  }
  tab[i] = new Entry(key, value);
  int sz = ++size;
  // 进行log(len)或者log(size)次探测未发现需要清理的slot 且达到需要rehash时 进行rehash
  if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
}

private void rehash() {
  // 先清理stale数据 如果有就可以减少size 
  expungeStaleEntries();
  // 如果size还是大于指定值 就继续执行扩容
  if (size >= threshold - threshold / 4)
    resize();
}
```





### 清除

```java
// remove之后 当前线程再调用get方法 就会再次触发initialValue的逻辑
public void remove() {
  ThreadLocalMap m = getMap(Thread.currentThread());
  if (m != null)
    m.remove(this);
}

private void remove(ThreadLocal<?> key) {
  Entry[] tab = table;
  int len = tab.length;
  int i = key.threadLocalHashCode & (len-1);
  for (Entry e = tab[i];
       e != null;
       e = tab[i = nextIndex(i, len)]) {
    // 找到相应的entry
    if (e.get() == key) {
      // key = null 设置为stale状态
      e.clear();
      // 清理该entry 以及向后遍历 清理stale entry和rehash entry 直到遇到null
      expungeStaleEntry(i);
      return;
    }
  }
}
```







