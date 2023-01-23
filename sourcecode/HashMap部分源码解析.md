# HashMap部分源码解析

[TOC]

> HashMap是一个基于哈希表的Map实现，允许null的key和value，不是线程安全的，哈希碰撞的解决方法是拉链法



从使用示例分析HashMap的主流程

```java
// 构造方法执行完成后 只是设置了数组长度和扩容阈值 
Map<Integer, String> map = new HashMap<>(4);
// 在第一次执行put方法时才会真正创建数组(懒加载)
map.put(1, "1");
map.put(2, "2");
map.put(3, "3");
// 使用误区：capacity指的是底层数组的大小 扩容阈值是capacity * loadfactor(默认0.75) 所以 4 * 0.75 = 3 
// 所以此时再put就会触发方法resize()
map.put(4, "4");
String s = map.get(1);
```



## 构造函数

```java
// 创建一个初始数组大小为4的哈希表
Map<Integer, String> map = new HashMap<>(4);

// 赋值默认的装载因子
this.loadFactor = 0.75f;
// tableSizeFor会返回 >= initialCapacity 的最小的2的倍数
// 这里是为了保证数组的大小为2的倍数 为了后面将取余运算简化为位运算中的与操作
this.threshold = tableSizeFor(initialCapacity);
```



## put方法

```java
map.put(1, "1");

return putVal(hash(key), key, value, false, true);

// 哈希值计算 这里的计算方式也跟数组下标的计算有关 index = hash & (capacity - 1) 因为capacity为2的倍数 所以 capacity - 1就是相当于一个低位全部为1的掩码
// 当capacity不高时 高位的bit的变化反映不到下标计算中 所以用位移 + 异或来使得下标计算时所用的哈希值 既反映了高位也反映了低位的变化
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
  			// tab：数组
  			// p：根据hash方法得出的哈希值计算出的数组下标对应的链表/红黑树头结点/null
  			// n: 数组大小
  			// i：数组下标 i = (n - 1) & hash 类似于掩码计算 只取低log2(n)位的bit
  			// 如n = 16 n - 1 = 15 二进制为1111B 
        Node<K,V>[] tab; Node<K,V> p; int n, i;
  			// 第一次执行put方法 在resize()中初始化数组
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
  			// 为空 没有哈希碰撞 所以直接赋值链表结点
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
          	// 存在哈希碰撞 遍历该bin中的结点
            Node<K,V> e; K k;
          	// 先判断头结点是不是 先比较hash值和key对象引用值 查看是否是同一个对象（进行值比较 比较快） 不匹配再进行.equals计算 
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
          	// 进入红黑树结点处理流程
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
              	// 遍历链表
                for (int binCount = 0; ; ++binCount) {
                  	// 遍历到链表尾 说明原来不存在该key-value
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                      	// 当链表长度 > 8 且数组长度 >= 64 时 链表->红黑树
                      	// 链表长度 > 8 数组长度 < 64 会执行resize()
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
          	// 之前存在该key-value
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                  	// 只替换value
                    e.value = value;
              	// afterNodexxx方法都是留给子类LinkedHashMap实现 模板方法模式
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
  			// 元素数量达到阈值 进行再散列扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
}
```



## resize方法

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // ... 计算newCap和newThr
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
      	// 再散列逻辑
      	// 遍历旧的数组
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
          	// 处理非空链表/二叉树
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
              	// 如果只有一个结点 直接移动
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                  	// 不只一个结点
                  	// loHead指向的链表表示是不需要移动位置的元素
                    Node<K,V> loHead = null, loTail = null;
                  	// hiHead指向的链表表示是需要移动位置的元素 且移动的offset为oldCap
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                  	// 遍历原链表 构造两个新的链表
                    do {
                        next = e.next;
                      	// 如oldCap = 16 这个就是判断 某个位是否为1 因为newCap = oldCap << 1
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

## get方法 

```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 遍历链表
          	do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

