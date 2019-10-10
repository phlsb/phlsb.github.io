---
layout: post
title:  "Java集合框架特征详解"
categories: Java基础
tags:  Java
author: 彭浩
---

* content
{:toc}

# ArrayList

源码简单，去看看就知道，基本的数组操作，下面就关键特点进行一下描述：

（1）数据结构与特性：数组、允许存入null元素、非线程安全，初始容量为10，

（2）扩容：1.5倍扩容，扩容过程为
    
    A. 先计算新数组容量 
    B、创建新的数组（这里在copyOf函数中通过反射创建的）
    C、进行数组元素之间的复制（这里调用System.arraycopy函数进行），值得注意的是ArrayList的扩容过程是先判断扩容然后进行元素的添加，与ConcurrentLinkedList就不同，这样有个缺点，当进行扩容后很长一段时间没有新的元素插入，那就会造成内存浪费。 

（3）方法上的特点（没啥特点，就扩容和数组复制有点特色）：

    A. add(object)方法，直接在当前尾部添加元素，没啥特别的
    B. add(index, object)，这就牵涉到数组元素的移动，依然使用System.arraycopy函数即可
    C. remove(index)，需要将删除位置元素置空来帮助垃圾回收，同时依然牵涉到数组元素移动
    D. remove(object)，该函数直接遍历整个数组来查找对应位置的元素，要分为是否为null元素的查找与普通元素的查找，同时牵涉到数组元素移动

# LinkedList

源码同样简单，基本的链表应用，下面就关键特点进行一下描述：

（1）数据结构与特性：双链表、节点类Node、头尾节点first与last、允许存入null元素，非线程安全，所有与下标相关的操作都是线性时间的，只有在头尾的插入和删除操作才是常数时间的，  

（2）方法上的特点（没啥特点，就node函数这个小trick而已）：

    A. add(e)，直接插入尾部，没什么特色，链表的插入操作
    B. add(index,e)，插入到指定位置需要先找到指定位置，找节点的位置使用node函数，该函数根据LinkedList的size属性判断该位置与头近还是与尾近一些，进而要么从头开始遍历找指定位置元素或者从尾开始遍历找指定位置元素。同时分为是否在尾部添加还是在中间添加（Linklast、LinkBefore）
    C. remove(index)，同样采用node函数找到位置对象，然后修改引用
    D. remobe(object)，这次需要直接根据传入的对象是否为null来遍历链表查找删除元素，然后修改引用

# ArrayDeque

源码同样简单，基本的数组应用，下面就关键特点进行一下描述：

（1）数据结构与特性：循环数组、头尾指针head（指向第一个有效元素位置）与tail（指向最后一个有效元素的下一个位置）、不允许存放null元素、非线程安全，初始容量为8

（2）扩容：插入后使用head == tail来判断数组是否满，扩容为2倍扩容，扩容过程依然为，A. 计算容量 B. 创建新数组 C. 使用System.arraycopy进行数组元素的复制（采用的是一半的复制两次，先复制head右半的元素，然后复制左半的元素，这样就使得每次扩容后head与tail指针复位了）  

（3）操作上的特点：  

    head与tail指针位置都可以加减，因为该类可以在任何地方插入和删除元素，由于是循环数组，为了避免下标越界问题，所以采用(head-1)&(length-1)，由于数组容量满足power of two的特性，所以length-1二进制表示全为1，它解决了如下两种边界情况，当tail为最后一个元素位置，那么加上1之后从二进制位来看数组长度位全为0，按位与上数组长度位全1就变为0，所以直接表示0，而当head是第一个元素位置，那么减去1之后为-1，而-1的二进制补码就是全1，这样按位与上数组长度位就表示最后一个元素，这样循环下标就实现了。而在0-length-1之间是单纯作为取余操作

（4）方法上的特点（没啥特点，就特殊操作与扩容比较特殊）：  

    A. addFirst、addLast等都很基本，判断下标是否越界以及是否需要扩容即可  
    B. pollFirst、pollLast，仅仅置空该位置元素然后修改head与tail即可


# TreeMap与TreeSet（红黑树相关）

（1）数据结构与特性：红黑树（采用非严格的平衡来换取增删节点时旋转次数的降低，相比AVL树在增删节点时要比红黑树次数多，总的来说红黑树调节平衡要比AVL树旋转次数少）、containsKey、get、put、remove操作的时间复杂度都为O(logn)、实现SortedMap接口（会按照key的大小顺序进行排序，可以是自然顺序，也可以是Comparator比较器的顺序）、非线程安全、可以使用Collections.synchronizedSortedMap来获取线程安全的TreeMap

（2）操作的特点

```java
A. 左旋，代码如下（手写）
private void rotateLeft(Entry<K, V> p){
    if(p != null){
        Entry<K, V> r = p.right;
        //修改挂枝节点的引用
        p.right = r.left;
        if(r.left != null){
            r.left.parent = p;
        }
        //修改父节点的引用
        r.parent = p.parent;
        if(p.parent == null){
            root = r;
        }else if(p.parent.left == p){
            p.parent.left = r;
        }else{
            p.parent.right = r;
        }
        //修改主节点引用
        p.parent = r;
        r.left = p;
    }
}


B. 右旋，代码如下（手写）
private void rotateRight(Entry<K, V> p){
    if(p != null){
        Entry<K, V> l = p.left;
        //修改挂枝节点引用
        p.left = l.right;
        if(l.right != null){
            l.right.parent = p;
        }
        //修改父亲节点引用
       l.parent = p.parent; 
       if(p.parent == null){
           root = l;
       }else if(p.parent.left == p){
           p.parent.left = l;
       }else{
           p.parent.right = l;
       }
       //修改主节点引用
       l.right = p;
       p.parent = l;
    }
}

C. 前驱节点，代码如下（手写）
static <K, V> Entry<K, V> predecessor(Entry<K, V> t){
    //在左子树中找最右边的
    if(t == null){
        return null;
    }else if(t.left != null){
        Entry<K, V> p = t.left;
        while(p.right != null){
            p = p.right;
        }
       return p; 
    }else{
        //在父节点中找第一次向右走的父节点
        Entry<K, V> parent = t.parent;
        Entry<K, V> child = t;
        while(parent != null && parent.left == child){
            child = parent;
            parent = parent.parent;
        }
        return parent;
    }    
}

D. 后继节点，代码如下（手写）
static <K, V> Entry<K, V> successor(Entry<K, V> t){
    if(t == null){
        return null;
    //找右子树中最左边的那个节点
    }else if(t.right != null){
        Entry<K, V> p = t.right;
        while(p.left != null){
            p = p.left;
        }
        return p;
    //找父节点中第一次向左走的父节点    
    }else{
        Entry<K, V> parent = t.parent;
        Entry<K, V> child = t;
        while(parent != null && parent.right == child){
            child = parent;
            parent = parent.parent;
        }
        return parent;
    }
}

E. 删除节点（手写）
static <K, V> Entry<K, V> delete(Entry<K, V> p){
    if(p == null){
        return null;
    }
    if(p.left != null && p.right != null){
        Entry<K, V> s = success(p);
        p.key = s.key;
        p.value = s.value;
        p = s;
    }
    //这一步表示当前p为删除节点，且要么没有子树、要么只有一颗子树
    Entry<K, V> replacement = p.left != null ? p.left : p.right;
    //若replacement不为空，表示有一颗子树，此时删除节点，修改引用
    if(replacement != null){
        replacement.parent = p.parent;
        if(p.parent == null){
            root = replacement;
        }else if(p.parent.left == p){
            p.parent.left = replacement;
        }else{
            p.parent.right = replacement;
        }
        p.left = p.right = p.parent = null;  
    }else if(p.parent == null){
        root = null;
    //若replacement为空，则表示没有子树的情况      
    }else{
        if(p.parent != null){
            if(p.parent.left == p){
                p.parent.left = null;
            }else if(p.parent.right == p){
                p.parent.right = null;
            }
            p.parent = null;
        }
    }
}

F. fixAfterInsertion，红黑树插入函数的实现
private void fixAfterInsertion(Entry<K, V> x){
    //先染红色，为什么染红色，后面会解释
    x.color = RED;
    //退出条件为x为null或者x为root，或者x颜色不为红
    while(x != null && x != root && x.color == RED){
        //对称情况，先看左边，影响的节点为x、p、g、b
        if(parentOf(x) == leftOf(parentOf(parentOf(x)))){
            Entry<K, V> y = rightOf(parentOf(parentOf(x)));
            //情况一：叔父节点为红色，将叔父节点与父节点染为黑色，祖父节点染为红色
            if(colorOf(y) == RED){
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                //设置x为祖父节点，向上继续调整
                x = parentOf(parentOf(x));
            }else{
                //情况三：当叔父节点为黑色，而x作为p的右孩子，先对p进行左旋
                if(rightOf(parentOf(x)) == x){
                    x = parentOf(x);//左旋后，x节点将变为子节点
                    rotateLeft(x);    
                }
                //情况二：当叔父节点为黑色，而x作为p的左孩子，对g进行右旋即可
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateRight(parentOf(parentOf(x)));
            }
        //对称情况，现在在右边    
        }else{
            Entry<K, V> y = leftOf(parentOf(paretOf(x)));
            //情况四：叔父节点为红
            if(colorOf(y) == RED){
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            }else{
                //情况六：叔父节点为黑色，x为p的左孩子
                if(leftOf(parentOf(x)) == x){
                    x = parentOf(x);
                    rotateRight(x);
                }
                //情况五：x为p的右孩子
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    setColor(root, BLACK);
}

G. fixAfterDeletion，红黑树删除函数的实现，该函数入参是删除节点的子节点，在没有子节点的情况下就是删除节点，且颜色可红和黑，这个不用管，因为情况讨论仅涉及其兄弟节点及兄弟节点的子节点
private void fixAfterDeletion(Entry<K, V> x){
    while(x != root && colorOf(x) != RED){
        //对称情况，当在左半边时
        if(leftOf(parentOf(x)) == x){
            Entry<K, V> sib = rightOf(parentOf(x));
            //情况四：当sib节点为红色，需要迭代
            if(colorOf(sib) == RED){
                setColor(parentOf(x), RED);
                setColor(sib, BLACK);
                rotateLeft(parentOf(x));
                sib = rightOf(parentOf(x));
            }
            //情况三：sib节点为黑色，并且左右子节点均为黑色
            if(colorOf(leftOf(sib)) == BLACK && corlorOf(rightOf(sib)) == BLACK){
                setColor(sib, RED);
                x = parentOf(x);
            }else{
                //情况二：sib节点为黑色，并且其左子节点为红色
                if(colorOf(leftOf(sib)) == RED){
                    setColor(sib, RED);
                    setColor(leftOf(sib), BLACK);
                    rotateRight(sib);
                    sib = rightOf(parentOf(x));
                }
                //情况一：sib节点为黑色，并且右子节点为红色
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(rightOf(sib), BLACK);
                rotateLeft(parentOf(x));
                x = root;//退出循环
            }
        //右半边情况        
        }else{
            Entry<K, V> sib = leftof(parentOf(x));
            //情况八，sib为红
            if(colorOf(sib) == RED){
                setColor(parentOf(x), RED);
                setColor(sib, BLACK);
                rotateRight(parentOf(x));
                sib = leftOf(parentOf(x));
            }
            //情况七：sib为黑，左右子节点均为黑
            if(colorOf(leftOf(sib)) == BLACK && colorOf(rightOf(sib)) == BLACK){
                setColor(sib, RED);
                x = parentOf(x);
            }else{
                //情况六：sib为黑，右子节点为红
                if(colorOf(rightof(sib)) == RED){
                    setColor(sib, RED);
                    setColor(rightOf(sib), BLACK);
                    rotateLeft(sib);
                    sib = parentOf(x);
                }
                //情况五：sib为黑，左子节点为红
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(leftOf(sib), BLACK);
                rotateRihgt(parentOf(x));
                x = root;
            }
        }
    }
    setColor(x, BLACK);
}
```

## 红黑树实现？

### 红黑树为什么比AVL插入效率要高?

首先要明白为什么插入节点一定要红色，而非黑色，因为若插入黑色节点违反红黑树最后一个性质，需要旋转来达到平衡，而若是染成红色，可能可以不进行红黑树结构的调整，若父节点为黑色的话。那为什么插入效率高：  
（1）一方面，红黑树的黑色节点至少为红色节点的两倍，若插入节点的父节点为黑色，那么不用调整，效率就上去了  
（2）另一方面，插入红色节点进行调整时，若其叔父节点为红色，那么可能仅需要进行颜色调整就可以使整个树平衡，故又一次减少旋转操作
鉴于上述两个原因，红黑树的插入效率要比AVL树好

### 红黑树的5个特点
（1）节点有只存在两种颜色，红和黑  
（2）根节点为黑  
（3）所有叶子节点均为黑色（NULL节点）  
（4）红色节点的父子节点必须全为黑色，即红色节点不能相邻  
（5）对于每个节点，从该节点到任意一个null节点的路路径包含的黑色节点个数相同

### 红黑树的查找，get方法，以treemap中的get方法为例，其调用getEntry来获取搜索节点，

（1）实际上就是二叉查找树的查找过程，直接使用键值key来比较，当键值key大于当前节点的key，则向右走，否则向左走  

（2）不同点在于比较键值的大小，要么使用自然顺序，要么使用自定义的顺序比较器，定义了当有自定义的Comparator时，调用自定义的key的compare方法来进行比较，而若没有自定义的Comparator，则将key强转为实现Comparable接口的对象，然后调用compareTo方法进行key值的比较

```java
final Entry<K,V> getEntry(Object key) {
    // Offload comparator-based version for sake of performance
    //对于自定义的comparator，将使用自定义的Comparator接口的compare方法来比较两个节点的达到，里面仅仅比较时采用的方法不同而已
    if (comparator != null)
        return getEntryUsingComparator(key);
    if (key == null)
        throw new NullPointerException();
    @SuppressWarnings("unchecked")
    //将key转型为可比较的k
    Comparable<? super K> k = (Comparable<? super K>) key;
    //从根节点开始遍历，使用实现Comparator接口的对象的compareTo方法比较两个元素的大小
    Entry<K,V> p = root;
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0)
            p = p.right;
        else
            return p;
    }
    return null;
}
```

### 红黑树的插入与修改操作，以treemap为例就是put方法，

（1）使用getEntry一样的逻辑先查找是否有对应key值相同的节点，若存在，直接调用setValue改变值即可，其中依然分为自然顺序的key值比较与自定义比较器顺序key的比较，同时在比较过程中记录一个parent值用于在找不到key相等对象需要后续插入的情况  
（2）若未找到key值相同的节点，则插入红黑树的尾部，根据com设置parent节点的引用，随后调用fixAfterInsertion方法调整红黑树的结构

```java
public V put(K key, V value) {
    //空判断，当树为空时，直接添加就好
    Entry<K,V> t = root;
    if (t == null) {
        compare(key, key); // type (and possibly null) check

        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    //使用自定义的比较器
    Comparator<? super K> cpr = comparator;
    //搜索过程，与getEntry干同样的事情，只是额外记录一个parent节点，若找到则调用setValue直接替换值
    if (cpr != null) {
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    //使用默认的比较器，一样是getEntry的逻辑
    else {
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    //若未找到，则添加节点到末尾，之前查找节点找了一个parent
    Entry<K,V> e = new Entry<>(key, value, parent);
    //根据com确定是左节点还是右节点
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;
    //添加该节点后调用fixAfterInsertion来调整红黑树的结构    
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```

### 红黑树的删除操作，以treemap为例的remove方法，其中调用了deleteEntry方法，在此之前依然是找到待删除的节点，逻辑与getEntry一样

（1）删除情况一：当删除的节点既有左子树又有右子树时，找到后继节点，然后交换该两个节点，从而变为删除情况二 

（2）删除情况二：删除的节点只有一颗子树或者没有子树，对于只有一颗子树的情况，直接修改相关引用后判断删除节点的颜色为黑色调用fixAfterDeletion调整红黑树结构，入参为删除节点的子节点。对于没有子树的情况，首先判断删除节点颜色为黑色调用fixAfterDeletion调整红黑树结构，入参为删除节点（作为虚拟节点，因为java中null代表空值，不能表示一个节点，所以复用删除节点作为虚拟的子节点传入），随后再修改相关引用

```java
private void deleteEntry(Entry<K,V> p) {
    modCount++;
    size--;

    // If strictly internal, copy successor's element to p and then make p
    // point to successor.
    //当删除节点P既有左子树也有右子树时，此时需要寻找后继节点，交换元素，然后转换成只有一颗子树或者无子树的节点删除情况
    if (p.left != null && p.right != null) {
        Entry<K,V> s = successor(p);//当前删除节点的后继节点
        p.key = s.key;
        p.value = s.value;
        p = s;//交换删除节点与其后继节点的元素并将当前删除节点的引用赋值为该后继节点，转换为情况A的删除
    } // p has 2 children

    // Start fixup at replacement node, if it exists.
    //要么只有一颗子树，要么没有左右子树
    Entry<K,V> replacement = (p.left != null ? p.left : p.right);
    //判空，若为空表明当前删除节点没有左右子树
    if (replacement != null) {
        // Link replacement to parent
        //修改父子节点之间的引用的常规操作
        replacement.parent = p.parent;
        if (p.parent == null)
            root = replacement;
        else if (p == p.parent.left)
            p.parent.left  = replacement;
        else
            p.parent.right = replacement;

        // Null out links so they are OK to use by fixAfterDeletion.
        p.left = p.right = p.parent = null;

        // Fix replacement
        //核心的删除节点若为黑色，将进行红黑树的结构调整，为什么这里是放在后面而非前面就进行调整
        if (p.color == BLACK)
            fixAfterDeletion(replacement);
    } else if (p.parent == null) { // return if we are the only node.
        root = null;
    } else { //  No children. Use self as phantom replacement and unlink.
        //左右子树都为空，则直接删除自己，并判断自身的颜色是否为黑色，若是则进行红黑树节点调整
        //注意：此处先进行红黑树的结构调整再对父子引用进行更改，原因是因为p节点首先将作为哨兵节点进行调整后才能删除
        if (p.color == BLACK)
            fixAfterDeletion(p);
        //若删除节点的父节点不为空，则要判断删除节点为父节点的左节点还是右节点，然后置空即可    
        if (p.parent != null) {
            if (p == p.parent.left)
                p.parent.left = null;
            else if (p == p.parent.right)
                p.parent.right = null;
            p.parent = null;
        }
    }
}
```
