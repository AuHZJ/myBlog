# Java学习笔记——`LinkedList`插入和删除速度真的比`ArrayList`快吗

> * 问：`LinkedList` 和 `ArrayList` 有什么区别？

> * 答：
> * `LinkedList` 实现了 `List` 和 `Deque` 接口，一般称为双向链表；
> * `ArrayList` 实现了` List` 接口，称为动态数组；
> * `LinkedList` 在插入和删除数据时效率更高；
> * `ArrayList` 在查找某个 `index` 的数据时效率更高；
> * `LinkedList` 比 `ArrayList` 需要更多的内存；

第二个区别的确没问题，但是少了条件，在某些条件下该结论是不成立的，先来看一个例子：

```java
public class CollectionTest {

    public static long addTime(List<Integer> list) {
        long start = System.currentTimeMillis(); // 起始时间
        for (int i = 0; i < 50000; ++i) {
            list.add(1); // ArrayList和LinkedList的add(element e)方法都是向末尾追加元素
        }
        long end = System.currentTimeMillis(); // 终止时间
        return end - start;
    }

    public static void main(String[] args) {
        System.out.println("ArrayList(add):" + addTime(new ArrayList<>()) + "ms"); // 测试ArrayList
        System.out.println("LinkedList(add):" + addTime(new LinkedList<>()) + "ms"); // 测试LinkedList
    }
}
```
插入`50000`个数据，运行结果为：
```java
ArrayList(get):3ms
LinkedList(get):2ms
```
问题不大，时间消耗差不多，因为时间复杂度都为`O(1)`，看看它们的源码：
`ArrayList.add(E e)源码:`
```java
/*
    ArrayList源码
 */
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!! 就是数组长度要+1
    elementData[size++] = e; // 这就是向末尾追加一个元素，显然时间复杂度为O(1)
    return true;
}
```
`LinkedList.add(E e)源码:`
```java
/*
    LinkedList源码
 */
public boolean add(E e) {
    linkLast(e); // 向末尾追加元素
    return true;
}

void linkLast(E e) {
    final Node<E> l = last; // last是list的末尾结点，由LinkedList类维护着
    final Node<E> newNode = new Node<>(l, e, null); // Node的创建格式   ->   Node<E>(前一个结点，数据，后一个结点)，这里的含义是newNode的上一个结点是l（last），后面结点为空，数据为e
    last = newNode; // 让last重新指向newNode，表示newNode是新的末尾结点
    if (l == null)
        first = newNode; // 如果l为空，即list为空，插入newNode后当然first = last = newNode
    else
        l.next = newNode; // l的下一个结点指向newNode
    size++;
    modCount++;
}
```

将上述例子中的`add`方法改为向指定位置添加元素，并事先在`list`里面添加`50000`个元素：
```java
public class CollectionTest {

    public static long addTime(List<Integer> list) {
        // 先让list里面有点东西，不然下面的add(i,1)就会变成在末尾添加元素，看不到效果
        for (int i = 0; i < 50000; ++i) { 
            list.add(1);
        }
        long start = System.currentTimeMillis(); // 起始时间
        for (int i = 0; i < 50000; ++i) {
            list.add(i, 1); // 注意此处的玄机
        }
        long end = System.currentTimeMillis(); // 终止时间
        return end - start;
    }

    public static void main(String[] args) {
        System.out.println("ArrayList(get):" + addTime(new ArrayList<>()) + "ms");
        System.out.println("LinkedList(get):" + addTime(new LinkedList<>()) + "ms");
    }
}
```
可以猜一猜结果，不出意外，`ArrayList`会快很多，结果如下：
```java
ArrayList(get):583ms
LinkedList(get):2118ms
```

究其原因，是因为`i`值越大，对`LinkedList`越不利，因为`LinkedList`需要遍历找到`i`这个位置，然后再插入值，这个代价是很大的，而`ArrayList`找`i`的位置是很快的，所以`i`越大，`LinkedList`的插入速度会越来越慢，而`ArrayList`的速度不会变化太大。

但是，`i`再大也有一定的上界的，我们可以将`i`设置为接近`list.size() - 1`，不要等于它，否则就是在末尾添加了，导致时间复杂度为`O（1）`，这里我们设置为`list.size() - 10`：
```java
public class CollectionTest {

    public static long addTime(List<Integer> list) {
        // 先让list里面有点东西，不然下面的add(i,1)就会变成在末尾添加元素，看不到效果
        for (int i = 0; i < 50000; ++i) {
            list.add(1);
        }
        long start = System.currentTimeMillis(); // 起始时间
        for (int i = 0; i < 50000; ++i) {
            list.add(list.size() - 10, 1); // 注意此处的玄机
        }
        long end = System.currentTimeMillis(); // 终止时间
        return end - start;
    }

    public static void main(String[] args) {
        System.out.println("ArrayList(get):" + addTime(new ArrayList<>()) + "ms");
        System.out.println("LinkedList(get):" + addTime(new LinkedList<>()) + "ms");
    }
}
```
结果：
```java
ArrayList(get):3ms
LinkedList(get):3ms
```
啊偶，不是说`i`越大速度越慢吗，怎么又变快了，但是如果把`i`变成`list.size() - 100`或者`list.size() - 1000`，你会发现，又变慢了，百思不得其解，究其源码，发现奥秘：
```java
/*
    LinkedList源码
 */
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index)); // 注意此处的node(index)方法
}

Node<E> node(int index) { // 找到索引为index的结点
    // assert isElementIndex(index);

    if (index < (size >> 1)) { // 如果index小于size/2，就从first开始遍历，即从头往后找
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {    // 如果index大于等于size/2，就从last开始遍历，即从尾向前找
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

原来是因为`LinkedList`优化了找指定结点的方式，靠近末尾结点就从末尾开始找，靠近开头就从开头开始找，这样就可以只遍历一半结点，这样我们很容易知道，当位置为`list`的中间时候将会是最费时的（需要遍历完一半），如下代码`LinkedList`的插入速度将会很慢：
```java
public class CollectionTest {

    public static long addTime(List<Integer> list) {
        // 先让list里面有点东西，不然下面的add(i,1)就会变成在末尾添加元素，看不到效果
        for (int i = 0; i < 50000; ++i) {
            list.add(1);
        }
        long start = System.currentTimeMillis(); // 起始时间
        for (int i = 0; i < 50000; ++i) {
            list.add(list.size() / 2, 1); // 注意此处的玄机
        }
        long end = System.currentTimeMillis(); // 终止时间
        return end - start;
    }

    public static void main(String[] args) {
        System.out.println("ArrayList(get):" + addTime(new ArrayList<>()) + "ms");
        System.out.println("LinkedList(get):" + addTime(new LinkedList<>()) + "ms");
    }
}
```
结果（根本不是一个级别的，你还敢说`LinkedList`比`ArrayList`快吗）：
```java
ArrayList(get):442ms
LinkedList(get):3629ms
```

同理，`remove`方法是一样的，当`index`的位置越靠中点，`LinkedList`会越慢：
```java
public class CollectionTest {

    public static long addTime(List<Integer> list) {
        // 先让list里面有点东西，不然下面的add(i,1)就会变成在末尾添加元素，看不到效果
        for (int i = 0; i < 50000; ++i) {
            list.add(1);
        }
        long start = System.currentTimeMillis(); // 起始时间
        for (int i = 0; i < 50000; ++i) {
            list.remove(list.size() / 2); // index为0会非常快
        }
        long end = System.currentTimeMillis(); // 终止时间
        return end - start;
    }

    public static void main(String[] args) {
        System.out.println("ArrayList(remove):" + addTime(new ArrayList<>()) + "ms");
        System.out.println("LinkedList(remove):" + addTime(new LinkedList<>()) + "ms");
    }
}
```
结果（根本不是一个级别的，你还敢说`LinkedList`比`ArrayList`快吗）：
```java
ArrayList(remove):210ms
LinkedList(remove):1428ms
```