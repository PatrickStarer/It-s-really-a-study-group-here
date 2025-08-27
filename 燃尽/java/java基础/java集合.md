## HashMap

HashMap 的底层结构在 Java 8 中是 “数组 + 链表 + 红黑树” 的组合，核心目的是平衡查询效率和空间利用率

1. 哈希桶数组（table）
   这是 HashMap 的主体，类型为 Node<K,V>[]（数组元素是 Node 节点）。数组的每个索引位置称为一个 “哈希桶”，用于存储哈希值相同的键值对。
    1. 数组长度（capacity）默认初始值为 16（DEFAULT_INITIAL_CAPACITY），且始终保持为 2 的幂（如
       16、32、64...），这是为了通过位运算优化哈希索引的计算。
    2. 数组长度可通过构造函数指定，但会被自动调整为最接近的 2 的幂。
2. 链表（Node 节点）
    1. 当多个键（key）的哈希值映射到同一个哈希桶（数组索引）时，会产生 “哈希冲突”。HashMap 通过 链地址法
       解决冲突：将哈希冲突的键值对以链表形式存储在同一个哈希桶中。
       链表节点由 Node 类实现，包含 hash（键的哈希值）、key、value、next（下一个节点的引用）四个字段。

3. 红黑树（TreeNode 节点）
   当链表长度超过阈值（TREEIFY_THRESHOLD = 8），且数组长度 ≥ MIN_TREEIFY_CAPACITY = 64 时，链表会被转换为红黑树（由 TreeNode
   类实现，继承自 Node）。
   红黑树是一种自平衡二叉搜索树（若数组长度 < 64，会先触发扩容而非转树，因为此时数组较小，扩容更能解决冲突），查找、插入、删除的时间复杂度为
   O(log n)，而链表为 O(n)。因此，当链表过长时，转为红黑树可显著提升查询效率。
   当红黑树节点数量减少到 UNTREEIFY_THRESHOLD = 6 时，会重新转为链表，避免红黑树维护成本过高。

## ConcurrentHashMap

结合 CAS 无锁操作 和 synchronized 细粒度锁 保证线程安全

1. 关键字段：
    1. volatile Node<K,V>[] table：哈希桶数组，volatile 保证数组引用的可见性（扩容或初始化时的修改对其他线程可见）。
    2. Node 节点的 value 和 next 字段被 volatile 修饰：保证读写的可见性（例如，链表节点的修改对其他线程可见）。
    3. sizeCtl：volatile 变量，用于控制初始化和扩容（负数表示正在初始化 / 扩容，正数表示扩容阈值，0 表示未初始化）。
2. 线程安全核心机制
    1. 初始化哈希桶数组：当 table 为 null 时，通过 CAS 原子更新 sizeCtl 标记 “正在初始化”，防止多线程重复初始化。
    2. 插入链表头节点：当哈希桶为空（table[i] == null）时，通过 CAS 直接插入新节点（casTabAt 方法），无需加锁。
    3. 扩容时的节点迁移：多线程协作扩容时，通过 CAS 标记已迁移的桶，避免重复处理。
       计数操作：通过 baseCount 和 CounterCell[] 数组（配合 CAS）实现并发环境下的元素数量统计（替代 Java 7 中 Segment 的
       count 累加）。
       （2）synchronized 细粒度锁（阻塞）
       当哈希桶不为空（存在哈希冲突）时，使用 synchronized 锁定桶的头节点（链表的第一个节点或红黑树的根节点），保证同一桶内的操作互斥：

3. put 操作流程：
   计算 key 的哈希值，确定哈希桶索引 i；
   若 table[i] 为 null，尝试用 CAS 插入新节点（成功则直接返回）；
   若 table[i] 不为 null，且处于扩容中（table[i] 是 ForwardingNode），则当前线程协助扩容；
   否则，对 table[i] 头节点加 synchronized 锁，然后遍历链表 / 红黑树：
   若找到相同 key 的节点，更新 value；
   若未找到，在链表尾部插入新节点（或红黑树插入）；
   插入后检查是否需要将链表转为红黑树（长度 ≥ 8）；
   释放锁，最后检查是否需要扩容。
   红黑树操作：红黑树的插入、删除等操作也通过锁定根节点保证线程安全，操作完成后会检查是否需要退化为链表（节点数 ≤ 6）。

4. get 操作的线程安全
   get 操作全程无锁，依赖 volatile 保证可见性：

哈希桶数组 table 是 volatile，确保能读到最新的数组引用；
Node 节点的 value 和 next 是 volatile，确保能读到其他线程修改后的最新值（例如，刚插入的节点或更新后的 value）。

5. 并发扩容机制（Java 8+ 关键优化）
   Java 8 中，ConcurrentHashMap 支持 多线程协作扩容，避免单线程扩容的性能瓶颈：
   触发扩容：当元素数量 size 超过扩容阈值（capacity × loadFactor）时，由当前线程启动扩容。
   扩容标记：通过 sizeCtl 标记扩容状态（sizeCtl = -1 表示初始化，sizeCtl = -(1 + 扩容线程数) 表示正在扩容）。
   多线程协作：其他线程在操作时若发现正在扩容（遇到 ForwardingNode 节点），会主动参与扩容（协助迁移节点），通过 CAS
   分配迁移范围，避免冲突。
   迁移过程：将旧数组的节点重新计算索引，迁移到新数组（长度为旧数组的 2 倍），迁移完成后用新数组替换旧数组。

## List
1. 什么是List？它与Collection、Set有什么区别？
定义：List是 Java 集合框架中Collection接口的子接口，用于存储有序、可重复的元素（允许 null 值）。
与Collection的关系：List继承自Collection，拥有Collection的所有方法（如add()、remove()、size()等），并额外提供了基于索引的操作（如get(int index)、set(int index, E element)）。
与Set的区别：
List：有序（元素插入顺序可保留）、可重复、支持索引访问。
Set：set也继承Collection，无序（部分实现如LinkedHashSet有序）、不可重复（依赖equals()和hashCode()判断）、不支持索引访问。
2. List的常用实现类有哪些？各自的底层数据结构是什么？
常见实现类包括ArrayList、LinkedList、Vector、Stack，底层结构差异如下：
ArrayList：底层是动态数组（数组长度可自动扩容），默认初始容量为 10（JDK1.8 及以上）。
LinkedList：底层是双向链表（JDK1.6 之前为循环双向链表，之后移除循环结构），每个节点包含prev（前驱指针）、next（后继指针）和元素值。
Vector：底层是动态数组，与ArrayList类似，但方法加了synchronized修饰，是线程安全的。
Stack：继承自Vector，底层是动态数组，实现了 “后进先出（LIFO）” 的栈结构（已被Deque替代，不推荐使用）。
3. ArrayList的扩容机制是怎样的？ 
ArrayList底层是数组，容量固定，当元素数量超过容量时会自动扩容，步骤如下：
初始容量：默认构造器创建的ArrayList在 JDK1.8 中是延迟初始化（首次add()时才分配容量 10）；若指定初始容量，则直接按指定值初始化。
触发扩容：当添加元素后，元素数量（size）超过当前容量（elementData.length）时，触发扩容。
计算新容量：
默认扩容为原容量的 1.5 倍（公式：oldCapacity + (oldCapacity >> 1)，位运算效率更高）。
若扩容后仍不足（如添加大量元素），则直接扩容至所需容量（Math.max(newCapacity, minCapacity)）。
复制元素：通过Arrays.copyOf()创建新数组，将原数组元素复制到新数组，原数组被垃圾回收。
优化建议：若已知大致元素数量，创建ArrayList时指定初始容量（如new ArrayList(1000)），可减少扩容次数，提升性能。
4. List的实现类中哪些是线程安全的？如何保证ArrayList线程安全？
线程安全的实现类：Vector（方法加synchronized）、Stack（继承Vector）、CopyOnWriteArrayList（JUC 包，写时复制机制）。
ArrayList线程不安全：多线程同时修改（如add/remove）可能导致数据不一致（如元素丢失）或抛出ConcurrentModificationException。
让ArrayList线程安全的方式：
使用Collections.synchronizedList(new ArrayList<>())：返回一个包装类，所有方法通过synchronized加锁（性能较低）。
使用CopyOnWriteArrayList（推荐）：读操作无锁（效率高），写操作（add/remove）时复制一份新数组，修改后替换原数组（适合 “读多写少” 场景）。
