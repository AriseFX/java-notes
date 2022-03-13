### HashMap put流程

```java
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
```
HashMap并没有直接使用使用hashcode作为hash值，而是做了一次运算
1.key为null，直接返回0
2.key不为null，将对象的hashcode值无符号右移16位，让高16位与hashcode进行异或运算，这样能得到一个hash值，这个值的低16位是hashcode的高低位异或的结果，而高位不变。
这是为了在数组长度较小的时候，hash值的低位也能足够离散，为什么异或能足够离散，异或运算结果每一位的1/0出现概率各是50%。与和或都不行，&运算使得结果向1靠近，|运算使得结果向0靠近
```

最重要就是putVal方法了
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
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
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

```
第一个判断是为了初始化主干数组，很好理解。但是这个resize比较有意思，后面再说。
第二个判断代表取模得到的槽没有数据，可以直接new一个Node
第二个判断的else中的逻辑就比较重要了
1.进到到这个else说明当前hash槽已经有了东西
2.如果旧key与新key相等，或者他们的hashcode相同且equal，那么可以说明是同一个key，那么要替换这个值。这一步也比较简单，无需扩容
3.如果不同，那么就要判断这个节点是链表还是树了
4.如果是链表，那么就要一步步找到链表的尾结点，在这个过程中，如果发现有相同的key，那么直接覆盖break。没有相同的key，那么就在尾结点的后面new一个Node，放置新key。
如果这个时候发现链表的长度大于8，那么调用treeifyBin将链表转成红黑树。
5.如果第3步的判断当前节点是树，那么就直接走红黑树的逻辑了
6.流程处理完成后，还有一个收尾的判断，那就是resize，如果发现
```
这个过程比较简单，但是有几个点值得注意:


```
1.高效取模。计算数组索引需要取模，hashmap的数组长度保持2的n次方，这种方式有好处，取模可以直接使用&来代替%，更加高效。

2.rehash。jdk1.8的hashmap的rehash非常有意思。因为hashmap扩容是扩容为原来的两倍，所以只需要把size左移一位，由于是2^n -1 ，所以转成二进制会在高位多一个1，所以只需要判断，多这个这个1与hashcode对应的位，&的结果是0还是1，如果是0，位置不变，如果是1，那么就把当前位置+old_size。所以扩容过后并不是都需要rehash，只需要判断hashcode的这一位是1还是0。所以呢
resize有这个判断if((e.hash & oldCap) == 0) ，就是为了获取那一位。

```

![hashmap_mode.png](img/hashmap_mode.png)

![hashmap_2.png](img/hashmap_2.png)

![hashmap_1.png](img/hashmap_1.png)

具体可以看看这篇:

https://tech.meituan.com/2016/06/24/java-hashmap.html