# Java集合篇

## 说说List,Set,Map三者的区别？

List(对付顺序的好帮手)： List接口存储一组不唯一（可以有多个元素引用相同的对象），有序的对象

Set(注重独一无二的性质): 不允许重复的集合。不会有多个元素引用相同的对象。

Map(用Key来搜索的专家): 使用键值对存储。Map会维护与Key有关联的值。两个Key可以引用相同的对象，但Key不能重复，典型的Key是String类型，但也可以是任何对象。

## HashMap 和 Hashtable 的区别

- `线程是否安全`： HashMap 是非线程安全的，HashTable 是线程安全的；HashTable 内部的方法基本都经过synchronized 修饰。（如果你要保证线程安全的话就使用 ConcurrentHashMap 吧！）；

- `效率`： 因为线程安全的问题，HashMap 要比 HashTable 效率高一点。另外，HashTable 基本被淘汰，不要在代码中使用它；

- `对Null key 和Null value的支持`： HashMap 中，null 可以作为键，这样的键只有一个，可以有一个或多个键所对应的值为 null。。但是在 HashTable 中 put 进的键值只要有一个 null，直接抛出 NullPointerException。

- `初始容量大小和每次扩充容量大小的不同` ： ①创建时如果不指定容量初始值，Hashtable 默认的初始大小为11，之后每次扩充，容量变为原来的2n+1。HashMap 默认的初始化大小为16。之后每次扩充，容量变为原来的2倍。②创建时如果给定了容量初始值，那么 Hashtable 会直接使用你给定的大小，而 HashMap 会将其扩充为2的幂次方大小（HashMap 中的tableSizeFor()方法保证，下面给出了源代码）。也就是说 HashMap 总是使用2的幂作为哈希表的大小,后面会介绍到为什么是2的幂次方。

- `底层数据结构`： JDK1.8 以后的 HashMap 在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为8）时，将链表转化为红黑树，以减少搜索时间。Hashtable 没有这样的机制。

HashMap 中带有初始容量的构造函数：

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
 public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```

下面这个方法保证了 HashMap 总是使用2的幂作为哈希表的大小。

```java
/**
 * Returns a power of two size for the given target capacity.
 */
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

## HashMap 和 HashSet区别？

HashSet 底层就是基于 HashMap 实现的。（HashSet 的源码非常非常少，因为除了 `clone() `、`writeObject()`、`readObject()`是 HashSet 自己不得不实现之外，其他方法都是直接调用 HashMap 中的方法。

| HashMap | HashSet|
| --- | --- |
| 实现了Map接口| 实现Set接口|
|存储键值对|仅存储对象|
|调用 put() 向map中添加元素|调用 add（）方法向Set中添加元素|
|HashMap使用键（Key）计算Hashcode|HashSet使用成员对象来计算hashcode值，对于两个对象来说hashcode可能相同，所以equals()方法用来判断对象的相等性
|

## HashMap的底层实现

### JDK1.8之前

JDK1.8 之前 HashMap 底层是 数组和链表 结合在一起使用也就是 链表散列。HashMap 通过 key 的 hashCode 经过扰动函数处理过后得到 hash 值，然后通过 (n - 1) & hash 判断当前元素存放的位置（这里的 n 指的是数组的长度），如果当前位置存在元素的话，就判断该元素与要存入的元素的 hash 值以及 key 是否相同，如果相同的话，直接覆盖，不相同就通过拉链法解决冲突。

所谓扰动函数指的就是 HashMap 的 hash 方法。使用 hash 方法也就是扰动函数是为了防止一些实现比较差的 hashCode() 方法 换句话说使用扰动函数之后可以减少碰撞。

JDK 1.8 HashMap 的 hash 方法源码:

JDK 1.8 的 hash方法 相比于 JDK 1.7 hash 方法更加简化，但是原理不变。

```java
static final int hash(Object key) {
    int h;
    // key.hashCode()：返回散列值也就是hashcode
    // ^ ：按位异或
    // >>>:无符号右移，忽略符号位，空位都以0补齐
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

JDK1.7的 HashMap 的 hash 方法源码.

```java
static int hash(int h) {
    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).

    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

相比于 JDK1.8 的 hash 方法 ，JDK 1.7 的 hash 方法的性能会稍差一点点，因为毕竟扰动了 4 次。

所谓 “拉链法” 就是：将链表和数组相结合。也就是说创建一个链表数组，数组中每一格就是一个链表。若遇到哈希冲突，则将冲突的值加到链表中即可。

![](https://img-blog.csdnimg.cn/20210124114013992.png)

### JDK1.8之后

相比于之前的版本， JDK1.8之后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为8）时，将链表转化为红黑树，以减少搜索时间。

![](https://img-blog.csdnimg.cn/202101241142486.png)

TreeMap、TreeSet以及JDK1.8之后的HashMap底层都用到了红黑树。红黑树就是为了解决二叉查找树的缺陷，因为二叉查找树在某些情况下会退化成一个线性结构。

## HashMap 的长度为什么是2的幂次方?

为了能让 HashMap 存取高效，尽量较少碰撞，也就是要尽量把数据分配均匀。我们上面也讲到了过了，Hash 值的范围值-2147483648到2147483647，前后加起来大概40亿的映射空间，只要哈希函数映射得比较均匀松散，一般应用是很难出现碰撞的。但问题是一个40亿长度的数组，内存是放不下的。所以这个散列值是不能直接拿来用的。用之前还要先做对数组的长度取模运算，得到的余数才能用来要存放的位置也就是对应的数组下标。这个数组下标的计算方法是“ (n - 1) & hash”。（n代表数组长度）。这也就解释了 HashMap 的长度为什么是2的幂次方。

### 这个算法应该如何设计呢？

我们首先可能会想到采用%取余的操作来实现。但是，重点来了：“取余(%)操作中如果除数是2的幂次则等价于与其除数减一的与(&)操作（也就是说 hash%length==hash&(length-1)的前提是 length 是2的 n 次方；）。” 并且 **采用二进制位操作 &，相对于%能够提高运算效率，这就解释了 HashMap 的长度为什么是2的幂次方。

## HashMap加载因子为什么是 0.75？

那加载因子为什么是 0.75 而不是 0.5 或者 1.0 呢？

首先如果加载因子比较大，那么扩容发生的频率就比较低，但是他浪费的空间比较小，不过发生hash冲突的几率就比较大，比如加载因子是1的时候，如果hashmap长度为128，那么可能hashmap的实际存储元素数量在64至128之间的时间段比较多，而这个时间段发生hash冲突就比较大，造成数组中其中一条链表较长，就会影响性能。

而当加载因子值比较小的时候，扩容的频率就会变高，因此会占用更多的空间，但是元素的存储就比较稀疏，发生哈希冲突的可能性就比较小，因此操作性能会比较高，比如设置成0.5，同样128长度的hashmap，当数量达到65的时候就会触发hashmap的扩容，扩容后长度为256，256里面只存储了65个似乎有点浪费了。

所以综合了以上情况就取了一个 0.5 到 1.0 的平均数 0.75 作为加载因子。

0.75与泊松分布的关系：当负载因子=0.75，带入泊松分布公式中，计算出来长度为8时，概率=0.00000006，这个0.00000006概率已经很小了，所以链表长度为8时，转换成红黑树

## 当有哈希冲突时，HashMap是如何查找并确认元素的？

通过昨天分析get方法应该已经比较清楚了，如果哈希冲突，还是找到table[index]的第一个Node，然后一个一个去比对链表中的key，key一致则找到，引用put流程图其中一部分如下图：

![](https://img-blog.csdnimg.cn/20210124115804339.png)

## JDK 1.8 HashMap 扩容时做了哪些优化？

在 JDK 1.7 中 HashMap 是以数组加链表的形式组成的，JDK 1.8 之后新增了红黑树的组成结构，当链表大于 8 并且容量大于 64 时，链表结构会转换成红黑树结构，即使在hashcode 完全相同极端情况下，由于红黑树的特点，查找某个特定元素，也只需要O(log n)的开销，而如果查找链表，时间复杂度会退化到 O(n)。

## HashMap 是线程安全的吗，为什么不是线程安全的？

通过例子来讲解这个不安全的过程，引用上一篇文章的例子，首先一个长度为8的map<Integer,V>，map已经存了key为1、9的两个值，他们都会在map的table[1]上。

1、首先线程1put一个key=17的值，当程序运行到判断key=9的下一个节点为null准备把key=17的值设置为它的下一个节点时，线程让出资源。

2、此时执行线程2，对mapput一个key=25的数据，这个数据正常作为key=9的next正常插入。

3、线程1再次拿到资源继续执行，设置key=9的next为key=17，这样线程2设置的key=25就被覆盖。

以上的流程示意图如下图：

![](https://img-blog.csdnimg.cn/20210124120005804.png)

关键代码在于HashMap的put代码，详解如下图：

![](https://img-blog.csdnimg.cn/20210124120046929.png)

## HashMap 多线程操作导致死循环问题?

主要原因在于1.8之前并发下的Rehash是头插法，会造成元素之间会形成一个循环链表。不过，jdk 1.8 后解决了这个问题，改成了尾插法，但是还是不建议在多线程下使用 HashMap,因为多线程下使用 HashMap 还是会存在其他问题比如数据丢失。并发环境下推荐使用 ConcurrentHashMap 。

### JDK7 头插法源码

![](https://img-blog.csdnimg.cn/20210124120145299.png)

### 头插法死循环原因

![](https://img-blog.csdnimg.cn/20210124120312653.png)

## map和set有什么区别，分别又是怎么实现的？

map和set都是C++的关联容器，其底层实现都是红黑树（RB-Tree）。由于 map 和set所开放的各种操作接口，RB-tree 也都提供了，所以几乎所有的 map 和set的操作行为，都只是转调 RB-tree 的操作行为。

### map和set区别在于：

（1）map中的元素是key-value（关键字—值）对：关键字起到索引的作用，值则表示与索引相关联的数据；Set与之相对就是关键字的简单集合，set中每个元素只包含一个关键字。

（2）set的迭代器是const的，不允许修改元素的值；map允许修改value，但不允许修改key。其原因是因为map和set是根据关键字排序来保证其有序性的，如果允许修改key的话，那么首先需要删除该键，然后调节平衡，再插入修改后的键值，调节平衡，如此一来，严重破坏了map和set的结构，导致iterator失效，不知道应该指向改变前的位置，还是指向改变后的位置。所以STL中将set的迭代器设置成const，不允许修改迭代器的值；而map的迭代器则不允许修改key值，允许修改value值。

（3）map支持下标操作，set不支持下标操作。map可以用key做下标，map的下标运算符[ ]将关键码作为下标去执行查找，如果关键码不存在，则插入一个具有该关键码和mapped_type类型默认值的元素至map中，因此下标运算符[ ]在map应用中需要慎用，const_map不能用，只希望确定某一个关键值是否存在而不希望插入元素时也不应该使用，mapped_type类型没有默认值也不应该使用。如果find能解决需要，尽可能用find。

## map底层为什么用红黑树实现？

1、红黑树：

红黑树是一种二叉查找树，但在每个节点增加一个存储位表示节点的颜色，可以是红或黑（非红即黑）。通过对任何一条从根到叶子的路径上各个节点着色的方式的限制，红黑树确保没有一条路径会比其它路径长出两倍，因此，红黑树是一种弱平衡二叉树，相对于要求严格的AVL树来说，它的旋转次数少，所以对于搜索，插入，删除操作较多的情况下，通常使用红黑树。

性质：

1. 每个节点非红即黑

2. 根节点是黑的;

3. 每个叶节点（叶节点即树尾端NULL指针或NULL节点）都是黑的;

4. 如果一个节点是红色的，则它的子节点必须是黑色的。

5. 对于任意节点而言，其到叶子点树NULL指针的每条路径都包含相同数目的黑节点;

2、平衡二叉树（AVL树）：

红黑树是在AVL树的基础上提出来的。

平衡二叉树又称为AVL树，是一种特殊的二叉排序树。其左右子树都是平衡二叉树，且左右子树高度之差的绝对值不超过1。

AVL树中所有结点为根的树的左右子树高度之差的绝对值不超过1。

将二叉树上结点的左子树深度减去右子树深度的值称为平衡因子BF，那么平衡二叉树上的所有结点的平衡因子只可能是-1、0和1。只要二叉树上有一个结点的平衡因子的绝对值大于1，则该二叉树就是不平衡的。

3、红黑树较AVL树的优点：

AVL 树是高度平衡的，频繁的插入和删除，会引起频繁的rebalance，导致效率下降；红黑树不是高度平衡的，算是一种折中，插入最多两次旋转，删除最多三次旋转。

所以红黑树在查找，插入删除的性能都是O(logn)，且性能稳定，所以STL里面很多结构包括map底层实现都是使用的红黑树。

## 如何决定使用 HashMap 还是 TreeMap？

对于在 Map 中插入、删除、定位一个元素这类操作，HashMap 是最好的选择，因为相对而言 HashMap 的插入会更快，但如果你要对一个 key 集合进行有序的遍历，那 TreeMap 是更好的选择。

## 请你说明一下TreeMap的底层实现？

TreeMap 的实现就是红黑树数据结构，也就说是一棵自平衡的排序二叉树，这样就可以保证当需要快速检索指定节点。

红黑树的插入、删除、遍历时间复杂度都为O(lgN)，所以性能上低于哈希表。但是哈希表无法提供键值对的有序输出，红黑树因为是排序插入的，可以按照键的值的大小有序输出。红黑树性质：

性质1：每个节点要么是红色，要么是黑色。

性质2：根节点永远是黑色的。

性质3：所有的叶节点都是空节点（即 null），并且是黑色的。

性质4：每个红色节点的两个子节点都是黑色。（从每个叶子到根的路径上不会有两个连续的红色节点）

性质5：从任一节点到其子树中每个叶子节点的路径都包含相同数量的黑色节点。

## ConcurrentHashMap线程安全的具体实现方式? 底层具体实现原理?

### JDK1.7的ConcurrentHashMap：

![](https://img-blog.csdnimg.cn/20210124114658748.png)

首先将数据分为一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据时，其他段的数据也能被其他线程访问。

ConcurrentHashMap 是由 Segment 数组结构和 HashEntry 数组结构组成。

Segment 实现了 ReentrantLock,所以 Segment 是一种可重入锁，扮演锁的角色。HashEntry 用于存储键值对数据。

java static class Segment extends ReentrantLock implements Serializable { }

一个 ConcurrentHashMap 里包含一个 Segment 数组。Segment 的结构和HashMap类似，是一种数组和链表结构，一个 Segment 包含一个 HashEntry 数组，每个 HashEntry 是一个链表结构的元素，每个 Segment 守护着一个HashEntry数组里的元素，当对 HashEntry 数组的数据进行修改时，必须首先获得对应的 Segment的锁。

### JDK1.8的ConcurrentHashMap

![](https://img-blog.csdnimg.cn/20210202232133778.png)

**Node：**

作为ConcurrentHashMap中最核心、最重要的内部类，Node担负着重要角色：key-value键值对。所有插入ConCurrentHashMap的中数据都将会包装在Node中。定义如下：

在Node内部类中，其属性value、next都是带有volatile的。同时其对value的setter方法进行了特殊处理，不允许直接调用其setter方法来修改value的值。最后Node还提供了find方法来赋值map.get()。

**TreeNode：**

在 HashMap 中，其核心的数据结构是链表。而在 ConcurrentHashMap 中，如果链表的数据过长会转换为红黑树来处理。通过将链表的节点包装成 TreeNode 放在 TreeBin 中，然后经由 TreeBin 完成红黑树的转换。

**Tree：**

TreeBin 不负责键值对的包装，用于在链表转换为红黑树时包装 TreeNode 节点，用来构建红黑树。

转换红黑树的源码：

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        //如果table.length<64 就扩大一倍 返回
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    //构造了一个TreeBin对象 把所有Node节点包装成TreeNode放进去
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        //这里只是利用了TreeNode封装 而没有利用TreeNode的next域和parent域
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    //在原来index的位置 用TreeBin替换掉原来的Node对象
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

ConcurrentHashMap取消了Segment分段锁，采用了 Node数组 + CAS + Synchronized来保证并发安全。数据结构跟HashMap1.8的结构类似，数组+链表/红黑树。Java 8在链表长度超过一定阈值（8）时将链表（寻址时间复杂度为O(N)）转换为红黑树（寻址时间复杂度为O(log(N))）

synchronized只锁定当前链表或红黑二叉树的首节点，这样只要hash不冲突，就不会产生并发，效率又提升N倍。

## ConcurrentHashMap 和 Hashtable 的区别?

- `底层数据结构`： JDK1.7的 ConcurrentHashMap 底层采用 分段的数组+链表 实现，JDK1.8 采用的数据结构跟HashMap1.8的结构一样，数组+链表/红黑二叉树。Hashtable 和 JDK1.8 之前的 HashMap 的底层数据结构类似都是采用 数组+链表 的形式，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的；

- `实现线程安全的方式（重要）`：

  ① 在JDK1.7的时候，ConcurrentHashMap（分段锁） 对整个桶数组进行了分割分段(Segment)，每一把锁只锁容器其中一部分数据，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率。 到了 JDK1.8 的时候已经摒弃了Segment的概念，而是直接用 Node 数组+链表+红黑树的数据结构来实现，并发控制使用 synchronized 和 CAS 来操作。（JDK1.6以后 对 synchronized锁做了很多优化） 整个看起来就像是优化过且线程安全的 HashMap，虽然在JDK1.8中还能看到 Segment 的数据结构，但是已经简化了属性，只是为了兼容旧版本；
  
  ② Hashtable(同一把锁) :使用 synchronized 来保证线程安全，效率非常低下。当一个线程访问同步方法时，其他线程也访问同步方法，可能会进入阻塞或轮询状态，如使用 put 添加元素，另一个线程不能使用 put 添加元素，也不能使用 get，竞争会越来越激烈效率越低。

## Arraylist 与 LinkedList 区别?

数据结构实现：ArrayList 是动态数组的数据结构实现，而 LinkedList 是双向链表的数据结构实现。

随机访问效率：ArrayList 比 LinkedList 在随机访问的时候效率要高，因为 LinkedList 是线性的数据存储方式，所以需要移动指针从前往后依次查找。

增加和删除效率：在非首尾的增加和删除操作，LinkedList 要比 ArrayList 效率要高，因为 ArrayList 增删操作要影响数组内的其他数据的下标。

综合来说，在需要频繁读取集合中的元素时，更推荐使用 ArrayList，而在插入和删除操作较多时，更推荐使用 LinkedList。

## ArrayList 与 Vector 区别呢?为什么要用Arraylist取代Vector呢？

线程安全：Vector 使用了 Synchronized 来实现线程同步，是线程安全的，而 ArrayList 是非线程安全的。

性能：ArrayList 在性能方面要优于 Vector。

扩容：ArrayList 和 Vector 都会根据实际的需要动态的调整容量，只不过在 Vector 扩容每次会增加 1 倍，而 ArrayList 只会增加 50%。

## 说一说 ArrayList 的扩容机制吧?

ArrayList是List接口的实现类，它是支持根据需要而动态增长的数组。java中标准数组是定长的，在数组被创建之后，它们不能被加长或缩短。这就意味着在创建数组时需要知道数组的所需长度，但有时我们需要动态程序中获取数组长度。ArrayList就是为此而生的。

因此，了解它的扩容机制对使用它尤为重要。

ArrayList扩容发生在add()方法调用的时候，下面是add()方法的源码：

```java
public boolean add(E e) {
   //扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

根据意思可以看出ensureCapacityInternal()是用来扩容的，形参为最小扩容量，进入此方法后：

```java
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
```

通过方法calculateCapacity(elementData, minCapacity)获取:

```java
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    //如果传入的是个空数组则最小容量取默认容量与minCapacity之间的最大值
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}
```

ensureExplicitCapacity方法可以判断是否需要扩容：

```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // 如果最小需要空间比elementData的内存空间要大，则需要扩容
    if (minCapacity - elementData.length > 0)
        //扩容
        grow(minCapacity);
}
```

接下来重点来了，ArrayList扩容的关键方法grow():

```java
private void grow(int minCapacity) {
    // 获取到ArrayList中elementData数组的内存空间长度
    int oldCapacity = elementData.length;
   // 扩容至原来的1.5倍
   int newCapacity = oldCapacity + (oldCapacity >> 1);
   // 再判断一下新数组的容量够不够，够了就直接使用这个长度创建新数组，
    // 不够就将数组长度设置为需要的长度
   if (newCapacity - minCapacity < 0)
       newCapacity = minCapacity;
   //若预设值大于默认的最大值检查是否溢出
   if (newCapacity - MAX_ARRAY_SIZE > 0)
       newCapacity = hugeCapacity(minCapacity);
   // 调用Arrays.copyOf方法将elementData数组指向新的内存空间时newCapacity的连续空间
   // 并将elementData的数据复制到新的内存空间
   elementData = Arrays.copyOf(elementData, newCapacity);
}
```

从此方法中我们可以清晰的看出其实ArrayList扩容的本质就是计算出新的扩容数组的size后实例化，并将原有数组内容复制到新数组中去。

**1：ArrayList 无参数构造器构造，现在 add 一个值进去，此时数组的大小是多少，下一次扩容前最大可用大小是多少？**

此处数组的大小是 1，下一次扩容前最大可用大小是 10，因为 ArrayList 第一次扩容时，是有默认值的，默认值是 10，在第一次 add 一个值进去时，数组的可用大小被扩容到 10 了。

**2：如果我连续往 list 里面新增值，增加到第 11 个的时候，数组的大小是多少？**

这里的考查点就是扩容的公式，当增加到 11 的时候，此时我们希望数组的大小为 11，但实际上数组的最大容量只有 10，不够了就需要扩容，扩容的公式是：oldCapacity + (oldCapacity>> 1)，oldCapacity 表示数组现有大小，目前场景计算公式是：10 + 10 ／2 = 15，然后我们发现 15 已经够用了，所以数组的大小会被扩容到 15。

**3：数组初始化，被加入一个值后，如果我使用 addAll 方法，一下子加入 15 个值，那么最终数组的大小是多少？**

第一题中我们已经计算出来数组在加入一个值后，实际大小是 1，最大可用大小是 10 ，现在需要一下子加入 15 个值，那我们期望数组的大小值就是 16，此时数组最大可用大小只有 10，明显不够，需要扩容，扩容后的大小是：10 + 10 ／2 = 15，这时候发现扩容后的大小仍然不到我们期望的值 16，这时候源码中有一种策略如下：

```java
// newCapacity 本次扩容的大小，minCapacity 我们期望的数组最小大小
// 如果扩容后的值 < 我们的期望值，我们的期望值就等于本次扩容的大小
if (newCapacity - minCapacity < 0)
    newCapacity = minCapacity;
```

所以最终数组扩容后的大小为 16。

## 请你说一说vector和list的区别

### ArrayList

1、实现原理：采用动态对象数组实现，默认构造方法创建了一个空数组

2、第一次添加元素，扩展容量为10，之后的扩充算法：原来数组大小+原来数组的一半

3、当插入、删除位置比较靠前时，与链表比较，不适合进行删除或插入操作

4、为了防止数组动态扩充次数过多，建议创建ArrayList时，给定初始容量

5、多线程中使用不安全，适合在单线程访问时使用，效率较高

### Vector

1、实现原理：采用动态数组对象实现，默认构造方法创建了一个大小为10的对象数组

2、扩充的算法：当增量为0时，扩充为原来的2倍，当增量大于0时，扩充为原来大小+增量

3、当插入、删除位置比较靠前时，与链表比较，不适合删除或插入操作

4、为了防止数组动态扩充次数过多，建议创建Vector时，给定初始容量

5、线程安全，适合在多线程访问时使用，效率较低

集合的使用注意：
若使用集合来存储多个不同类型的元素（对象），那么在处理时会比较麻烦
在实际开发中不建议这样使用，我们应该在一个集合中存储相同的类型对象

## Array 和 ArrayList 有何区别？

Array 可以存储基本数据类型和对象，ArrayList 只能存储对象。

Array 是指定固定大小的，而 ArrayList 大小是自动扩展的。

Array 内置方法没有 ArrayList 多，比如 addAll、removeAll、iteration 等方法只有 ArrayList 有。

### Iterator 和 ListIterator 有什么区别？

Iterator 可以遍历 Set 和 List 集合，而 ListIterator 只能遍历 List。

Iterator 只能单向遍历，而 ListIterator 可以双向遍历（向前/后遍历）。

ListIterator 从 Iterator 接口继承，然后添加了一些额外的功能，比如添加一个元素、替换一个元素、获取前面或后面元素的索引位置。

## 阐述ArrayList、Vector、LinkedList的存储性能和特性

1、ArrayList采用的是数组形式来保存对象的，这种方式将对象放在连续的位置中，所以最大的缺点就是插入删除时非常麻烦

2、LinkedList采用的将对象存放在独立的空间中，而且在每个空间中保存下一个链接的索引，但是缺点就是查找非常麻烦，要从第一个索引开始

3、ArrayList和Vector都是用数组方式存储数据，此数组元素数要大于实际的存储空间以便进行元素增加和插入操作，他们都允许直接用序号索引元素，但是插入数据元素涉及到元素移动等内存操作，所以索引数据快而插入数据慢

4、Vector使用了synchronized方法（线程安全），所以在性能上比ArrayList要差些


5、LinkedList使用双向链方式存储数据，按序号索引数据需要向前或向后遍历数据，索引索引数据慢，插入数据时只需要记录前后项即可，所以插入的速度快

## LinkedList的实现原理总结如下：


①数据存储是基于双向链表实现的。

②插入数据很快。先是在双向链表中找到要插入节点的位置index，找到之后，再插入一个新节点。 双向链表查找index位置的节点时，有一个加速动作：若index < 双向链表长度的1/2，则从前向后查找; 否则，从后向前查找。

③删除数据很快。先是在双向链表中找到要插入节点的位置index，找到之后，进行如下操作：node.previous.next = node.next;node.next.previous = node.previous;node = null 。查找节点过程和插入一样。

④获取数据很慢，需要从Head节点进行查找。

⑤遍历数据很慢，每次获取数据都需要从头开始。

## 快速失败 (fail-fast) 和安全失败 (fail-safe) 的区别是什么？

### 1、快速失败（fail-fast）

在用迭代器遍历一个集合对象时，如果遍历过程中对集合对象的内容进行修改（增加、删除、修改），则会抛出Concurrent Modification Exception.

`原理`：迭代器在遍历时直接访问集合中的内容，并且在遍历过程中使用一个modCount变量。集合在被遍历期间如果内容发生变化，就会改变modCount的值。每当迭代器使用hashNext()/next()遍历下一个元素之前，都会检测modCount变量是否为expectedmodCount值，是的话就返回遍历；否则抛出异常，终止遍历。

`注意`：这里异常的抛出条件是检测到modCount!=expectedmodCount这个条件。如果集合发生变化时修改modCount值刚好又设置为了expectedmodCount值，则异常不会抛出。因此，不能依赖于这个异常是否抛出而进行并发操作的编程，这个异常只建议用于检测并发修改的bug。

`场景`：java.util包下的集合类都是快速失败的，不能在多线程下发生并发修改（迭代过程中被修改）。

### 2、安全失败（fail-safe）

采用安全失败机制的集合容器，在遍历时不是直接在集合内容上访问的，而是先复制原有集合内容，在拷贝的集合上进行遍历。
原理：由于迭代时是对原集合的拷贝进行遍历，所以在遍历过程中对原集合所作的修改并不能被迭代器检测到，所以不会触发Concurrent Modification Exception。

`缺点`：基于拷贝内容的优点是避免了Concurrent Modification Exception,但同样地，迭代器并不能访问到修改后的内容，即：迭代器遍历的是开始遍历那一刻拿到的集合拷贝，在遍历期间原集合发生的修改迭代器是不知道的

`场景`：java.util.concurrent包下的容器都是安全失败，可以在多线程下并发使用，并发修改。

