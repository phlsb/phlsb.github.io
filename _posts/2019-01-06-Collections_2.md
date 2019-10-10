---
layout: post
title:  "Java集合框架特征详解（续）"
categories: Java基础
tags:  Java
author: 彭浩
---

* content
{:toc}

# HashMap与HashSet

（1）数据结构与特征：键值对映射，采用数组+链表+红黑树来实现，允许key与value都为null，不保证元素顺序，且在扩容时会进行重hash，在不同时刻，遍历的元素顺序可能不同、对于hashmap的遍历操作是比较耗时的，对于频繁迭代的场景，不宜将hashmap的初始容量设的过大。hashmap的初始容量为16

（2）关键参数：

    A. TREEIFY_THRESHOLD，值为8，表示链表中节点数目超过该阈值时将转化为红黑树（注意C的前提）  
    B. UNTREEIFY_THRESHOLD，值为6，表示红黑树节点数目不超过该数目时，将退化为链表  
    C. MIN_TREEIFY_CAPACITY，值为64，表示当总容量没有超过该值时，即使链表节点个数超过8，那么直接进行扩容，而不是进行树型化  
    D. loadFactor，负载因子，初始为0.75，当实际的节点数目超过loadFactor*capacity时，将进行自动扩容

（3）注意：自定义的对象放入HashMap，需要重写hashcode与equals

（4）方法剖析：  

简单说一下，在hashmap的构造函数中，对loadFactor进行初始化，要么是传入的值要么是默认的0.75，构造函数部分也会对threshold进行初始化，这是通过传入的initialCapacity，通过tableForSize函数计算一个最近的大于该容量且满足power of two的值

#### 查询的方法，get，从hashmap中获取对应键值的value值，而键值的定位是采用hashcode的hash值以及key值比较得来的，调用链如下，

（1）hash函数采用的是键值的hashcode的高16位与低16位进行异或运算，是简单性、性能的折中，  

（2）getNode方法，流程很简单，

    A. 使用hash值按位与上数组长度位定位桶
    B. 快速判断桶首节点是否为搜索节点，通过hash值、==与equals来判断，若是就可以退出了，若不是则进入C
    C. 根据首节点是链表节点还是树型节点对目标节点进行搜索，若是链表节点，则循环遍历即可，其中采用hash值、==与equals来判断是否相等，若是树型节点，则调用getTreeNode函数来在红黑树中查找对应目标键值，其中getTreeNode调用了find函数进行搜索过程，流程不像TreeMap的二叉查找树的搜索那样简单，要复杂很多。具体逻辑如下，
        a. 用hash值判断向左走还是向右走，若相等，则使用==和equals是否该节点与目标节点相等，若相等，则找到了该节点，可以退出了，但若不相等，则要继续遍历下去
        b. 在hash值相等的情况下，优先直接向不为空的子树走，若都不为空，则使用自定义的比较器来比较键值key的大小（强转键值类型使用comparableForClass和compareComparables）来确定方向
        c. 若无法使用键值的比较器，则强制先向右走找一遍，若未找到与目标键值相等的节点，则继续在该位置左走找一遍，直到都找完

特别注意：计算出的hash值也是32位的，而在定位是按位与上了一个数组长度位的，所以即使在同一个桶中，也不一定hash值就想等，所以第一道关卡就是hash值

```Java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {//键值key的hash值与数组长度减一按位与获得桶位置，找到链表的首节点
        //依据hash值、==、与equals来判断当前链表节点K值是否与入参key相等，相等则找到，可以返回了
        //在这里不进行循环相当于一次快速的判断第一个节点是否是我们想要找的节点
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        //若该链表不止一个节点，则需要继续寻找key值的节点    
        if ((e = first.next) != null) {
            //此处与1.7不同，在1.8中引入了红黑树的结构，判断当前链表是否已经进化为红黑树，若是红黑树结构，将按红黑树的结构查找key值节点
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            //否则，依然为链表结构，此时就一个个节点取出来，然后使用hash值、==与equals来进行比较查找key值节点    
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}

final TreeNode<K,V> getTreeNode(int h, Object k) {
    return ((parent != null) ? root() : this).find(h, k, null);
}

final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
    TreeNode<K,V> p = this;
    do {
        int ph, dir; K pk;
        TreeNode<K,V> pl = p.left, pr = p.right, q;
        //比较hash值，判断进入左边还是右边
        if ((ph = p.hash) > h)
            p = pl;
        else if (ph < h)
            p = pr;
        //hash值相等比较键值key是否相等，使用==和equals进行比较，相等则找到，不相等继续下面的判断    
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        //若红黑树当前节点有一个子节点为空，则直接继续另一个节点的遍历    
        else if (pl == null)
            p = pr;
        else if (pr == null)
            p = pl;
        //当没有子节点为空，此时使用自定义的比较器来比较键值，确定走哪边    
        else if ((kc != null ||
                  (kc = comparableClassFor(k)) != null) &&
                 (dir = compareComparables(kc, k, pk)) != 0)
            p = (dir < 0) ? pl : pr;
        //若采用自定义的比较器比较失败或者比较为相等，此时先强行走右边寻找   ， 
        else if ((q = pr.find(h, k, kc)) != null)
            return q;
        //否则，在右子树中未找到，到左边继续寻找    
        else
            p = pl;
    } while (p != null);
    return null;
}
```

#### put方法，put方法实现节点的修改与插入，源码调用链如下，牵涉到插入节点，所以会破坏链表和红黑树的结构，也牵涉到链表转化为红黑树的过程，更牵涉到扩容的过程，调用链如下，如下阐释几个非常重要的方法，

（1）putVal方法，是put方法调用的根方法，整体是实现hashmap的put操作的，由于hashmap在构造器初始化时并未定义桶数组，仅仅是赋值了两个变量，所以初始化桶数组的事情就被延迟到put函数中，所以流程为，  
    
    A. 桶数组为空，则先进行桶数组的初始化，调用resize扩容函数进行桶数组的初始化，后面介绍这个resize方法。
    B. 按照getNode的逻辑查找目标节点，使用hash值按位与数组长度位定位桶，并快速判断桶首节点是否为目标节点，若是则可以直接修改值退出了，若不是则进入C
    C. 依然是分为链表节点的遍历与树型节点的遍历，对于链表节点，若遍历到尾部还未发现目标节点相等的节点，则新建该键和值的节点，并赋值给尾节点的next引用，随后判断链表节点个数是否已经大于TREEIFY_THRESHOLD，超过该阈值调用treeifybin函数将链表转化为红黑树，该函数后面着重讲解，若为树型节点，则调用putTreeVal函数在红黑树中查找目标节点，若存在，则返回该节点并修改值即可，若不存在，则牵涉到红黑树节点的插入，后面详细介绍
    D. 在链表或者红黑树的put操作完成后，则还需要判断整个桶数组的节点数量是否已经超过threshold，超过则需要调用resize函数进行扩容。

（2）putTreeVal方法，是针对红黑树的put操作的函数，当检测到桶首节点为树型节点时，调用该方法进行红黑树的put操作，逻辑如下，

    A. 同样是内置了find函数的操作，在红黑树中查找节点，这里不再赘述，需要注意的是，除了find函数中的各种方向判断走向外（稍微不同的是在强制确定方向中先左再右），还加入了一个强制措施，对键值的类名称进行字典序比较来进行方向走向判断，当依然没有找到目标节点，那么就要进行红黑树节点的插入，这操作又与TreeMap中put操作一样了，首先创建这个节点并根据方向值来挂上父节点引用，不同的是hashmap的树型节点还有next、prev与parent引用，修改这几个引用，随后调用balanceInsertion进行红黑树结构的调整（这又与TreeMap的fixAfterInsertion是一样的）

（3）treeifyBin方法，用于链表大于TREEIFY_THRESHOLD阈值时链表转化为红黑树的函数，也要注意当整个桶数组总的节点个数小于MIN_TREEIFY_CAPACITY时，仅仅调用resize函数扩容，而非链表树型化，链表树型化的逻辑为，

    A. 判断当前桶数组容量是否还未超过阈值MIN_TREEIFY_CAPACITY，则调用resize进行扩容，而非链表树型化，resize后面再解释
    B. 根据链表节点创建树型节点，通过next引用将新创建的树型节点按照链表顺序串联起来，这样构造树型节点的链表后，调用treeify函数构建红黑树，该方法逻辑为下面C开始的步骤，
    C. treeify函数的逻辑实际上与putTreeVal函数一样，这里仅仅是从头开始遍历，从0构建红黑树而已，其中逻辑是首先选择首节点作为根节点，随后对于接下来的迭代的节点依然是走find的逻辑，利用hash值、左右子树非空、自定义比较器以及最终的强制方向判断tieBreakOrder来对目标节点进行定位，这里是为了判断当前节点是否已经构建过，随后发现没有构建过，根据方向值修改引用后，再调用balanceInsertion方法进行红黑树结构的调整即可。

（4）resize函数，用于扩容的函数，在三种情况下会调用该函数进行扩容，即初始化时、链表树型化时可能、总的节点数超过threshold时，hashmap的扩容牵涉到重hash，同时涉及到红黑树退化为链表的操作，逻辑为
    
    A. 因为扩容可能出现在桶数组还未初始化，以及桶数组已经初始化中任意一个阶段，所以扩容的第一步就是数组未初始化以及数组初始化的阶段的容量Cap以及扩容阈值thread的计算，对于初始化阶段的值，就设置为默认值，对于真实的扩容则设置为原来的2倍。
    B. 接下来节点迁移的过程，分为链表节点的迁移与树型节点的迁移，下面分两步C与D来分别阐释链表节点与树型节点的迁移，其中链表节点迁移就在resize中，而树型节点的迁移则调用的是split函数
    C. 链表节点的迁移，思想就是创建两个新的链表，因为即使是重hash，2倍扩容，所以该节点在新桶数组中的位置要么是原来的位置要么是加上一个原数组长度的位置，这两者是由节点的hash值按位与上原数组容量决定的（新数组长度位实际上就是原数组长度位左移一位，正好就是判断该位的hash值是否为1，进而在原节点位置基础上判断是否需要加上原数组长度）。

    所以在遍历桶数组的每个桶时，若桶中存在至少一个节点，则创建一个原位置的低位链表与新位置的高位链表，然后遍历原链表，根据节点hash值按位与上新数组长度最高位来将节点以单链表尾插的方式加入这两个链表，随后将这两个链表直接挂在新桶数组对应位置即可。
    
    D. 树型节点的迁移，与链表节点的迁移思想几乎一样，同样是建立两个链表，而之前链表树型化时，调用treeifyBin函数会先将链表节点使用next引用连成单链表，这里就体现出作用了，直接遍历该next引用组成的单链表，依然是使用尾插的方式根据节点hash值按位与新数组长度最高位来确定挂载的链表，但是注意在最后将这两个链表赋值给桶时，还需要判断当前树型节点的个数，若小于等于UNTREEIFY_THRESHOLD，则需要退化为链表调用untreeify，否则还是调用treeify函数构建为红黑树。untreeify非常简单，将树型节点退化为链表节点即可，而treeify在上面已经解释过了，就是红黑树的构建。最后就是将这两个链表赋值给桶即可
    
    所以说，这个链表与红黑树的中间链表是非常重要的，在扩容中不仅遍历原树节点有用，而且在退化成链表节点或者构造成红黑树都非常重要。

```Java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //若当前容器为空，则扩容，
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //使用hash值与数组容量定位目标桶    
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        //查找，首先使用hash值与key值快速判断桶的第一个节点是否key值与目标相等，若相等，则找到目标修改节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //若快速查找失败，则判断头节点是否为树型节点，若是，则进行红黑树的查找与插入操作，调用putTreeVal函数    
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //若头节点为链表型节点，则遍历链表节点查找key值，若未查找到，则需要进行插入，其中binCount是链表节点数的计数器
            for (int binCount = 0; ; ++binCount) {
                //当遍历到链表尾部，表明未查找到目标key值节点，此时新建一个链表节点，并判断binCount是否大于TREEIFY_THRESHOLD，若大于
                //则需要将链表树型化，调用treeifyBin函数进行链表树型化，注意该函数内部还会判断一次MIN_TREEIFY_CAPACITY判断是否真实需要树型化
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //遍历过程就是扫描节点的hash值和key值判断是否找到目标节点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //在找到修改节点e后，更改其value值即可
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //若此时容器容量大于threshold，则需要进行扩容
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}

final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                               int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false;
    TreeNode<K,V> root = (parent != null) ? root() : this;
    //遍历红黑树节点，准备查找
    for (TreeNode<K,V> p = root;;) {
        int dir, ph; K pk;
        //根据hash值选择遍历方向
        if ((ph = p.hash) > h)
            dir = -1;
        else if (ph < h)
            dir = 1;
        //当发现相等的键值，则可以退出函数进行返回    
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        //hash值相等，而键值不相等的情况下，使用自定义的compareTo方法进行遍历方向的确认    
        else if ((kc == null &&
                  (kc = comparableClassFor(k)) == null) ||
                 (dir = compareComparables(kc, k, pk)) == 0) {
            if (!searched) {
                TreeNode<K,V> q, ch;
                searched = true;
                //当自定义的比较器比较失败或为0，则先在左子树中查找目标键值的节点，否则在右子树中查找
                if (((ch = p.left) != null &&
                     (q = ch.find(h, k, kc)) != null) ||
                    ((ch = p.right) != null &&
                     (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            //若之前的所有查找都还未比较出遍历方向，则调用该函数强行确认一个方向
            dir = tieBreakOrder(k, pk);
        }

        TreeNode<K,V> xp = p;
        //判断是继续遍历还是可以插入了
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            //修改相关节点的引用
            Node<K,V> xpn = xp.next;
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
            if (dir <= 0)
                xp.left = x;
            else
                xp.right = x;
            xp.next = x;
            x.parent = x.prev = xp;
            if (xpn != null)
                ((TreeNode<K,V>)xpn).prev = x;
            //进行红黑树的调整    
            moveRootToFront(tab, balanceInsertion(root, x));
            return null;
        }
    }
}

final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //若当前容器的容量还未超过MIN_TREEIFY_CAPACITY，则进行扩容而非将链表树型化
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    //否则，链表长度已经超过8，将进行链表树型化，使用hash值与长度减一来定位该链表    
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;//记录双链表的头节点和遍历节点的前继节点
        do {
            //按照链表节点创建新的树型节点
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        //将单链表构造成树型节点的双链表后，将头结点复制给该桶的元素
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}

final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    //遍历在treefyBin函数中形成的TreeNode类型的双链表，从而取出节点构建真正的红黑树
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        //以当前桶中的第一个节点作为根节点（后面会进行调整的）
        if (root == null) {
            x.parent = null;
            x.red = false;
            root = x;
        }
        else {
            //取出遍历节点数据，并准备插入红黑树
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            //从根节点开始对红黑树进行插入位置的查找
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;
                K pk = p.key;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                          //当hash值相等时，需要进行键值的比较来搜索插入位置
                          //该方法返回键值k的Class对象，且是实现了Comparable的Class对象，若未实现则返回null
                          //首先获取该对象的Class对象，就可以查看键值类型的接口索引集合，从而判断该对象是否实现了Comparable接口，
                          //实现则返回该类Class对象，否则返回null
                          (kc = comparableClassFor(k)) == null) ||
                          //当对象实现了Comparable接口，即可进行键值的比较，
                         (dir = compareComparables(kc, k, pk)) == 0)
                    //若键值比较失败或相等，则调用该函数，该函数是最后一次比较，必定返回-1或者1，而不会返回0，
                    //其实现通过判断k与key值的类名，并比较大小，若属于相同的类，则调用本地方法为两个对象生成hashcode值，，再进行比较，相等的话返回-1 
                    dir = tieBreakOrder(k, pk);
                //到此为止，该链表节点插入红黑树的位置以及作为其父节点的左子节点还是右子节点已经确认了
                TreeNode<K,V> xp = p;
                //判断当前插入节点x是否已经插入过
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    //节点插入红黑树
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    //插入后进行而插入的平衡操作，旋转节点
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    moveRootToFront(tab, root);
}

final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    //获取原数组的容量
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //获取原数组的临界值
    int oldThr = threshold;
    int newCap, newThr = 0;
    //分情况计算新的容量与新的临界值，包括当数组为空时的计算
    if (oldCap > 0) {
        //情况三：当原数组容量不为0时，表明是真实的扩容，
        //当原数组容量已经大于MAXIMUM_CAPACITY时，则无法扩容，赋值新的临界值为MAXIMUM_CAPACITY，然后直接返回原数组
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //若原数组容量还未超过MAXIMUM_CAPACITY，且将原数组容量扩容为两倍后在DEFAULT_INITIAL_CAPACITY以及MAXIMUM_CAPACITY之间，计算新数组容量为原数组容量的两倍，且临界值为原临界值的两倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    //情况二：当原数组为空，且已经初始化了初始的临界值，则初始化新数组容量为初始化的临界值（猜测该值同样在初始化时就保证了power-of-two）
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    //情况一：当原数组为空，且并未初始化临界值，此时初始化新的数组容量为DEFAULT_INITIAL_CAPACITY，并通过loadFactor计算新的临界值    
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    //根据新的容量创建新的数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        //遍历原数组
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;//原数组桶中的元素
            //当当前桶不为空
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                //当该桶中只有一个节点，直接根据新的数组容量进行重映射即可，相当于快速判断
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                //若当前桶中有多个节点，而节点类型为树型节点，说明将要进行红黑树节点的迁移    
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                //若当前桶中有多个节点，而节点类型为链表类型的节点，说明将要进行链表节点的迁移
                else { // preserve order
                    //lo前缀的指针与hi前缀的指针分别表示原数组中应该映射到新数组的前半部分的节点与应该映射到新数组的后半部分的节点
                    //此处的逻辑是先建出链表，然后直接将头节点赋值给对应桶即可，这里分为左半部分与右半部分
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;//记录后继节点
                        //该值很优雅，使用（e.hash & oldCap）原理是：原数组中该元素的位置为hash值按位与上原数组容量减一的值，而新数组容量为原来的两倍，
                        //那么按位与上新数组容量减一实际上就是hash值多增加了一位用于定位桶，从十进制上看就是原数组桶的位置加上原数组容量作为偏移就是
                        //在新数组桶的位置，但是注意一定是该位的hash值要为1才是原数组桶位置加上原数组容量，否则还是原位置
                        //所以总结来说就是：当该值为1，表明新添加的计算位为1，那么桶的定位位置为原位置加上原数组容量，而为0时，则在新数组中还是原位置
                        if ((e.hash & oldCap) == 0) {
                            //左半部分，记录头节点loHead，loTail作为新链表的遍历节点，使用尾插的方式
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        //右半部分，
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    //链表建立完毕后，将头节点分别赋值给对应的桶
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

final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
    TreeNode<K,V> b = this;
    // Relink into lo and hi lists, preserving order
    //构建链表的节点指针
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;//链表节点的计数
    //根据红黑树的next指针进行节点的遍历，这个next指针是在链表转化为树型结构的treeifyBin函数中构建的
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        next = (TreeNode<K,V>)e.next;
        e.next = null;
        //根据（e.hash & oldCap）对节点进行分类，一类为位置不变的节点，一类为加上原数组容量偏移的位置的节点
        if ((e.hash & bit) == 0) {
            //原位置的部分链表构建
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;//原位置节点计数
        }
        else {
            //加上原数组容量偏移的位置部分链表构建
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;//新位置部分的节点计数
        }
    }
    //将原位置部分的节点链表赋值给对应的桶，注意要判断是否进行将红黑树节点退化为链表节点的操作还是要将新构建的链表树形化
    //所以这里看出双链表是链表节点与树型化节点的中间层，用于树型化和退化为链表的中间层
    if (loHead != null) {
        //若节点数小于等于UNTREEIFY_THRESHOLD，则需要将树型节点退化为链表节点，调用untreeify函数即可
        if (lc <= UNTREEIFY_THRESHOLD)
            //退化操作即将双链表转化为单链表，且将树型节点转化为链表节点即可
            tab[index] = loHead.untreeify(map);
        else {
            //若节点数还是大于UNTREEIFY_THRESHOLD，则将新的树型节点构建的双链表转化为红黑树，调用treeify函数
            tab[index] = loHead;
            if (hiHead != null) // (else is already treeified)
                loHead.treeify(tab);
        }
    }
    //新位置部分与上操作一样
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}

final Node<K,V> untreeify(HashMap<K,V> map) {
    Node<K,V> hd = null, tl = null;
    for (Node<K,V> q = this; q != null; q = q.next) {
        Node<K,V> p = map.replacementNode(q, null);
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    return hd;
}
```

#### remove方法，
该方法实现桶中节点的删除，实际上调用的是removeNode方法进行核心实现，其实put方法就是一个find方法和一个resize方法及红黑树的调整是核心不同的，多的勉强的不同即链表树型化和树型节点退化（退化发生在扩容）的过程，删除节点同样逃不过put的核心find逻辑，首先走find的逻辑，通过hash值定位桶后，分别针对是链表节点还是树型节点进行find逻辑查找，链表就是遍历使用hash、==、equals判断即可，红黑树就是二叉搜索树的查找按照dir方向值走调用getTreeNode方法，查找到该节点后（似乎若开启了匹配value值，则最后还需要判断是否value相等才能删除），对于链表来说直接修改引用即可，对于红黑树，则调用removeTreeNode来进行搜索到的节点进行删除，该方法的逻辑对应的是TreeMap的红黑树删除操作fixAfterDeletion的实现。这里不再赘述

```Java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}

final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                          boolean movable) {
    int n;
    if (tab == null || (n = tab.length) == 0)
        return;
    int index = (n - 1) & hash;
    TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
    TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
    if (pred == null)
        tab[index] = first = succ;
    else
        pred.next = succ;
    if (succ != null)
        succ.prev = pred;
    if (first == null)
        return;
    if (root.parent != null)
        root = root.root();
    if (root == null || root.right == null ||
        (rl = root.left) == null || rl.left == null) {
        tab[index] = first.untreeify(map);  // too small
        return;
    }
    TreeNode<K,V> p = this, pl = left, pr = right, replacement;
    if (pl != null && pr != null) {
        TreeNode<K,V> s = pr, sl;
        while ((sl = s.left) != null) // find successor
            s = sl;
        boolean c = s.red; s.red = p.red; p.red = c; // swap colors
        TreeNode<K,V> sr = s.right;
        TreeNode<K,V> pp = p.parent;
        if (s == pr) { // p was s's direct parent
            p.parent = s;
            s.right = p;
        }
        else {
            TreeNode<K,V> sp = s.parent;
            if ((p.parent = sp) != null) {
                if (s == sp.left)
                    sp.left = p;
                else
                    sp.right = p;
            }
            if ((s.right = pr) != null)
                pr.parent = s;
        }
        p.left = null;
        if ((p.right = sr) != null)
            sr.parent = p;
        if ((s.left = pl) != null)
            pl.parent = s;
        if ((s.parent = pp) == null)
            root = s;
        else if (p == pp.left)
            pp.left = s;
        else
            pp.right = s;
        if (sr != null)
            replacement = sr;
        else
            replacement = p;
    }
    else if (pl != null)
        replacement = pl;
    else if (pr != null)
        replacement = pr;
    else
        replacement = p;
    if (replacement != p) {
        TreeNode<K,V> pp = replacement.parent = p.parent;
        if (pp == null)
            root = replacement;
        else if (p == pp.left)
            pp.left = replacement;
        else
            pp.right = replacement;
        p.left = p.right = p.parent = null;
    }

    TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);

    if (replacement == p) {  // detach
        TreeNode<K,V> pp = p.parent;
        p.parent = null;
        if (pp != null) {
            if (p == pp.left)
                pp.left = null;
            else if (p == pp.right)
                pp.right = null;
        }
    }
    if (movable)
        moveRootToFront(tab, r);
}
```

# PriorityQueue

（1）数据结构及特征：采用数组实现二叉堆，且是小顶堆的实现。因为牵涉到元素的大小，所以与TreeMap一样可以采用节点本身的自然顺序以及自定义的比较器来比较大小。查询元素是常数时间，但add、remove等都是O(logn)的时间复杂度、不能插入null元素（除了ArrayDeque就是这个了）。初始容量为11。  

（2）扩容：扩容依然是三个步骤，1）计算新的容量值 2）创建新的数组 3）调用System.arraycopy函数进行数组元素的复制即可，PrioritQueue的扩容稍微有点不一样，在数组容量较小时，即64以内，则进行双倍扩容，实际上是双倍之上加上2，而在容量超过64时，将进行1.5倍扩容。  

（3）方法剖析，主要介绍peek、add、offer、poll方法

##### peek，不用讲，就是堆顶元素，即数组的0号元素即可，

#### add方法，底层直接调用offer方法，不同在于插入失败的处理，add是抛出异常，而offer则返回false。添加元素是在二叉堆的尾部添加，一方面，既然是使用数组实现二叉堆，添加元素势必牵涉到扩容过程，而添加节点的元素值可能破坏小顶堆的结构，故需要进行siftUp操作来修复二叉堆的结构，逻辑如下，

    A. 先判断原数组容量，是否已满，若满，则调用grow函数进行扩容，这个处理逻辑与ConcurrentHashMap是一样的，先将数组添加满了之后才进行扩容，这就避免了之前实现的扩容中还未满就进行扩容的浪费存储空间的缺陷。grow函数在扩容特征中已经进行了讲解。
    B. 直接调用siftUp函数进行小顶堆结构的调整，默认是节点直接插入到尾部，其中siftUp函数根据是否自定义了比较器分别调用两个相同逻辑仅仅是比较方法调用不同的函数siftUpUsingComparator与siftUpComparable来进行，核心逻辑阐释如下面C步骤
    C. 直接迭代向上进行调整，直到找到比添加节点元素值小的节点停止，在父节点中（父节点计算是(i-1)/2）找到比当前添加的值小的节点即可停止迭代

```Java
public boolean offer(E e) {
    //放入元素为null，则抛出空指针异常
    if (e == null)
        throw new NullPointerException();
    modCount++;
    //数组容量检查
    int i = size;
    if (i >= queue.length)
        //容量不够，进行扩容
        grow(i + 1);
    size = i + 1;
    //第一个元素直接插入
    if (i == 0)
        queue[0] = e;
    else
        //在叶子节点插入并调整二叉堆结构
        siftUp(i, e);
    return true;
}

private void siftUp(int k, E x) {
    if (comparator != null)
        siftUpUsingComparator(k, x);
    else
        siftUpComparable(k, x);
}

private void siftUpUsingComparator(int k, E x) {
    //k表示空出来的位置，迭代使用插入元素与其父节点的大小相比，当发现比当前父节点小，则
    //会将空位向上挪，实现上就是将该空位赋值为父节点，直到发现比当前父节点大，此时空位就是
    //插入元素放入的位置
    while (k > 0) {
        //父节点位置
        int parent = (k - 1) >>> 1;
        Object e = queue[parent];
        //只有找到父节点元素比插入元素小才能退出循环迭代
        if (comparator.compare(x, (E) e) >= 0)
            break;
        queue[k] = e;
        k = parent;
    }
    queue[k] = x;
}

private void siftUpComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>) x;
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = queue[parent];
        if (key.compareTo((E) e) >= 0)
            break;
        queue[k] = e;
        k = parent;
    }
    queue[k] = key;
}
```

#### remove方法，无参的remove方法用于删除队首元素，而有参的remove方法删除任意位置的元素，所以删除队首元素是删除任意节点方法的特例方法，删除给定对象的remove方法的核心逻辑就是要先找到该删除节点的位置，随后删除该节点并进行小顶堆结构的调整，逻辑为

    A. 调用indexOf查找删除节点所在的位置
    B. 调用removeAt方法删除节点并调整小顶堆的结构，逻辑阐释如下C和D步骤
    C. 快速判断删除的节点是否为尾节点，若是则不用调整堆结构了，若不是，则使用堆尾节点作为填补节点进行堆结构调整，若堆尾节点经过若干次的siftDown操作后填补到堆结构中的空位，那么调整过程就结束了，而若是堆尾节点第一次就填补了删除节点的空位，那么还需要进行siftUp操作，具体需要看siftDown操作的逻辑，为D
    D. siftDown函数，依然根据是否有自定义的比较器来分别调用逻辑相同仅比较方法不同的向下调整堆结构的方法，这里直接结束逻辑，
        a. 根据完全二叉树的特点（节点数目除以2即可）判断删除位置是否是叶子节点，若是，则直接使用堆尾节点填补即可，注意这也算堆尾节点直接填补删除节点的情况，在removeAt函数中还要调用siftUp函数，
        b. 若不是叶子节点，取当前空位节点（初始为删除节点）的两个子节点中的小者，与堆尾节点比较，取其中的小者作为当前空位节点元素，若采用了堆尾节点填补，则可退出循环，而若是子节点中的小者填补当前空位，则空位下移至子节点中小者，迭代继续，直到堆尾节点填补空位或者已经到达叶子节点。
    E. 判断是否是堆尾节点直接填补的删除节点空位，若是，则调用siftUp函数向上调整堆结构，因为可能堆尾节点要比其父节点更小，而在非直接填补删除节点空位的情况，则子节点一定要比堆尾节点小从而防止父节点比子节点大的情况出现。

```Java
public boolean remove(Object o) {
    //遍历数组，使用equals方法查找对应位置
    int i = indexOf(o);
    if (i == -1)
        return false;
    else {
        //删除该位置节点
        removeAt(i);
        return true;
    }
}

private E removeAt(int i) {
    // assert i >= 0 && i < size;
    modCount++;
    //第一种情况：删除的是最后一个元素，不用调整二叉堆结构，直接将该位置置为null
    int s = --size;
    if (s == i) // removed last element
        queue[i] = null;
    else {
        //第二种情况：删除的是除最后一个节点以外的任意一个节点，
        E moved = (E) queue[s];//以最后一个节点作为参照
        queue[s] = null;//置空最后一个节点
        //该方法相当于删除位置为一个空位，现在需要其他节点来填这个空位，这个空位要么由
        //其子节点中小者填，要么由尾节点填，当子节点中小者比尾节点大，则尾节点将填充到
        //空位中，若一直使用子节点中小者来迭代填充，可能到达倒数第二层的叶子节点导致违反
        //完全二叉树的性质，此时再用尾节点来填充就晚了，故在发现尾节点比两个子节点都小时
        //就使用尾节点来填充空位，若一直子节点的小者都小于尾节点，则退出后依然适用尾节点
        //填充，注意，在第一个空位就发现尾节点可以填充到该空位，还将进行sitfUp的操作，因为
        //此时尾节点的大小不确定就比删除节点的父节点大，但是一定比删除节点的子节点小。而若是
        //第二个空位或者更后面的空位被尾节点填充，则一定满足比空位的父节点大，因为会经过比较
        //前一次没有用尾节点填充就是因为子节点中有更小者，这样尾节点一定比这个小者要大，而该
        //小者此时作为空位的父节点，所以就满足尾节点填充空位一定比父节点大，而此次又比空位的
        //子节点小，故满足
        siftDown(i, moved);
        //所以，此处判断该删除位置的空位是否是被尾节点填充了，若被尾节点填充了，说明是第一个
        //空位就被尾节点填充，此时还需要调用siftUp方法向上进行调整
        if (queue[i] == moved) {
            //就是一直找父节点比当前节点小的节点
            siftUp(i, moved);
            if (queue[i] != moved)
                return moved;
        }
    }
    return null;
}

private void siftDown(int k, E x) {
    if (comparator != null)
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x);
}

private void siftDownUsingComparator(int k, E x) {
    //根据完全二叉树的size来判断删除节点是否是最后一层的节点
    int half = size >>> 1;
    //当是最后一层节点直接退出循环，将尾节点赋值到该删除位置空位即可，若是其他层的节点，则进入循环
    while (k < half) {
        //获取左子节点下标
        int child = (k << 1) + 1;
        //获取左子节点值
        Object c = queue[child];
        //获取右子节点下标
        int right = child + 1;
        //获取子节点中的小者以及下标
        if (right < size &&
            comparator.compare((E) c, (E) queue[right]) > 0)
            c = queue[child = right];
        //若尾节点小于该小者，则可退出循环，并直接赋值该空位为尾节点，注意当是第一个空位就被尾节点赋值，则还需进行向上调整，在外面    
        if (comparator.compare(x, (E) c) <= 0)
            break;
        //否则，使用小者来填充空位，并将空位下移到小者之前的位置    
        queue[k] = c;
        k = child;
    }
    queue[k] = x;
}
```