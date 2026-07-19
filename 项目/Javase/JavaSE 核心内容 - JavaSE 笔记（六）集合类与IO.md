![image-20221004131436371](https://s2.loli.net/2022/10/04/SkAn9RQpqC4tVW5.png)

# 集合类与IO

前面我们已经把基础介绍完了，从这节课开始，我们就正式进入到集合类的讲解中。

## 集合类

集合类是Java中非常重要的存在，使用频率极高。集合其实与我们数学中的集合是差不多的概念，集合表示一组对象，每一个对象我们都可以称其为元素。不同的集合有着不同的性质，比如一些集合允许重复的元素，而另一些则不允许，一些集合是有序的，而其他则是无序的。

![image-20220930233059528](https://s2.loli.net/2022/09/30/ZWxPduaYGgRzmNO.png)

集合类其实就是为了更好地组织、管理和操作我们的数据而存在的，包括列表、集合、队列、映射等数据结构。从这一块开始，我们会从源码角度给大家讲解（先从接口定义对于集合需要实现哪些功能开始说起，包括这些集合类的底层机制是如何运作的）不仅仅是教会大家如何去使用。

集合跟数组一样，可以表示同样的一组元素，但是他们的相同和不同之处在于：

1. 它们都是容器，都能够容纳一组元素。

不同之处：

1. 数组的大小是固定的，集合的大小是可变的。
2. 数组可以存放基本数据类型，但集合只能存放对象。
3. 数组存放的类型只能是一种，但集合可以有不同种类的元素。

### 集合根接口

Java中已经帮我们将常用的集合类型都实现好了，我们只需要直接拿来用就行了，比如我们之前学习的顺序表：

```java
import java.util.ArrayList;   //集合类基本都是在java.util包下定义的

public class Main {
    public static void main(String[] args) {
        ArrayList<String> list = new ArrayList<>();
        list.add("树脂666");
    }
}
```

当然，我们会在这一部分中认识大部分Java为我们提供的集合类。所有的集合类最终都是实现自集合根接口的，比如我们下面就会讲到的ArrayList类，它的祖先就是Collection接口：

![image-20250708234355615](https://s2.loli.net/2025/07/08/6tHdU5Fcawh3juN.png)

这个接口定义了集合类的一些基本操作，我们来看看有哪些方法：

```java
public interface Collection<E> extends Iterable<E> {
    //-------这些是查询相关的操作----------

   	//获取当前集合中的元素数量
    int size();

    //查看当前集合是否为空
    boolean isEmpty();

    //查询当前集合中是否包含某个元素
    boolean contains(Object o);

    //返回当前集合的迭代器，我们会在后面介绍
    Iterator<E> iterator();

    //将集合转换为数组的形式
    Object[] toArray();

    //支持泛型的数组转换，同上
    <T> T[] toArray(T[] a);

    //-------这些是修改相关的操作----------

    //向集合中添加元素，不同的集合类具体实现可能会对插入的元素有要求，
  	//这个操作并不是一定会添加成功，所以添加成功返回true，否则返回false
    boolean add(E e);

    //从集合中移除某个元素，同样的，移除成功返回true，否则false
    boolean remove(Object o);


    //-------这些是批量执行的操作----------

    //查询当前集合是否包含给定集合中所有的元素
  	//从数学角度来说，就是看给定集合是不是当前集合的子集
    boolean containsAll(Collection<?> c);

    //添加给定集合中所有的元素
  	//从数学角度来说，就是将当前集合变成当前集合与给定集合的并集
  	//添加成功返回true，否则返回false
    boolean addAll(Collection<? extends E> c);

    //移除给定集合中出现的所有元素，如果某个元素在当前集合中不存在，那么忽略这个元素
  	//从数学角度来说，就是求当前集合与给定集合的差集
  	//移除成功返回true，否则false
    boolean removeAll(Collection<?> c);

    //Java8新增方法，根据给定的Predicate条件进行元素移除操作
    default boolean removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);
        boolean removed = false;
        final Iterator<E> each = iterator();   //这里用到了迭代器，我们会在后面进行介绍
        while (each.hasNext()) {
            if (filter.test(each.next())) {
                each.remove();
                removed = true;
            }
        }
        return removed;
    }

    //只保留当前集合中在给定集合中出现的元素，其他元素一律移除
  	//从数学角度来说，就是求当前集合与给定集合的交集
  	//移除成功返回true，否则false
    boolean retainAll(Collection<?> c);

    //清空整个集合，删除所有元素
    void clear();


    //-------这些是比较以及哈希计算相关的操作----------

    //判断两个集合是否相等
    boolean equals(Object o);

    //计算当前整个集合对象的哈希值
    int hashCode();

    //与迭代器作用相同，但是是并行执行的，我们会在下一章多线程部分中进行介绍
    @Override
    default Spliterator<E> spliterator() {
        return Spliterators.spliterator(this, 0);
    }

    //生成当前集合的流，我们会在后面进行讲解
    default Stream<E> stream() {
        return StreamSupport.stream(spliterator(), false);
    }

    //生成当前集合的并行流，我们会在下一章多线程部分中进行介绍
    default Stream<E> parallelStream() {
        return StreamSupport.stream(spliterator(), true);
    }
}
```

可以看到，在这个接口中对于集合相关的操作，还是比较齐全的，那么我们接着就来看看它的实现类。

### List列表

首先我们需要介绍的是List列表（线性表），线性表支持随机访问，相比之前的Collection接口定义，功能还会更多一些。首先介绍ArrayList，我们已经知道，它的底层是用数组实现的，内部维护的是一个可动态进行扩容的数组，也就是我们之前所说的顺序表，跟我们之前自己写的ArrayList相比，它更加的规范，并且功能更加强大，同时实现自List接口。

![image-20250708234355615](https://s2.loli.net/2025/07/08/6tHdU5Fcawh3juN.png)

List是集合类型的一个分支，它的主要特性有：

* 是一个有序的集合，插入元素默认是插入到尾部，按顺序从前往后存放，每个元素都有一个自己的下标位置
* 列表中允许存在重复元素

在List接口中，定义了列表类型需要支持的全部操作，List继承自SequencedCollection接口，此接口是Java 21新增接口，此前List接口直接继承自Collection接口（老版图片放在迭代器部分）此接口额外定义了获取第一个元素、最后一个元素，生成反向集合视图等功能，统一了所有有序集合的操作定义，这些内容我们会放在后面进行介绍。

可以看到在List接口中，很多地方重新定义了一次Collection和SequencedCollection接口中定义的方法，虽然没有任何修改，但是这样做是为了更加明确方法的具体功能，当然，为了直观，我们这里就省略掉：

```java
//List是一个有序的集合类，每个元素都有一个自己的下标位置
//List中可插入重复元素
//针对于这些特性，扩展了Collection接口中一些额外的操作
public interface List<E> extends Collection<E> {
    ...
   	
    //将给定集合中所有元素插入到当前结合的给定位置上（后面的元素就被挤到后面去了，跟我们之前顺序表的插入是一样的）
    boolean addAll(int index, Collection<? extends E> c);

    ...

   	//Java 8新增方法，可以对列表中每个元素都进行处理，并将元素替换为处理之后的结果
    default void replaceAll(UnaryOperator<E> operator) {
        Objects.requireNonNull(operator);
        final ListIterator<E> li = this.listIterator();  //这里同样用到了迭代器
        while (li.hasNext()) {
            li.set(operator.apply(li.next()));
        }
    }

    //对当前集合按照给定的规则进行排序操作，这里同样只需要一个Comparator就行了
    @SuppressWarnings({"unchecked", "rawtypes"})
    default void sort(Comparator<? super E> c) {
        Object[] a = this.toArray();
        Arrays.sort(a, (Comparator) c);
        ListIterator<E> i = this.listIterator();
        for (Object e : a) {
            i.next();
            i.set((E) e);
        }
    }

    ...

    //-------- 这些是List中独特的位置直接访问操作 --------

   	//获取对应下标位置上的元素
    E get(int index);

    //直接将对应位置上的元素替换为给定元素
    E set(int index, E element);

    //在指定位置上插入元素，就跟我们之前的顺序表插入是一样的
    void add(int index, E element);

    //移除指定位置上的元素
    E remove(int index);


    //------- 这些是List中独特的搜索操作 -------

    //查询某个元素在当前列表中的第一次出现的下标位置
    int indexOf(Object o);

    //查询某个元素在当前列表中的最后一次出现的下标位置
    int lastIndexOf(Object o);


    //------- 这些是List的专用迭代器 -------

    //迭代器我们会在下一个部分讲解
    ListIterator<E> listIterator();

    //迭代器我们会在下一个部分讲解
    ListIterator<E> listIterator(int index);

    //------- 这些是List的特殊转换 -------

    //返回当前集合在指定范围内的子集
    List<E> subList(int fromIndex, int toIndex);

    ...
}
```

可以看到，在List接口中，扩展了大量列表支持的操作，其中最突出的就是直接根据下标位置进行的增删改查操作。而在ArrayList中，底层就是采用数组实现的，跟我们之前的顺序表思路差不多（如果需要学习老版本请观看22年旧版JavaSE视频课程）：

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
		
    //默认的数组容量
    private static final int DEFAULT_CAPACITY = 10;

    ...

    //存放数据的底层数组，这里的transient关键字我们会在后面I/O中介绍用途
    transient Object[] elementData;

    //记录当前数组元素数的
    private int size;

   	//这是ArrayList的其中一个构造方法
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];   //根据初始化大小，创建当前列表
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
  
    //这是ArrayList的无参构造方法
  	public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
  
  	...
      
   	public boolean add(E e) {
        modCount++;   //用于后期判断是否出现并发异常，比如遍历时修改或是多线程操作（暂时忽略）
        add(e, elementData, size);  //调用内部私有辅助方法实现插入操作，因为是尾插，index直接写size就行
        return true;  //直接返回真插入成功
    }
  
    private void add(E e, Object[] elementData, int s) {
        if (s == elementData.length)   //首先判断长度是否超出当前内部数组容量
            elementData = grow();   //超出那么就得扩容，扩容会对类的elementData进行重新赋值，下面介绍
        elementData[s] = e;   //接着正常插入元素即可
        size = s + 1;  //让size自增
    }
  
    public void add(int index, E element) {
        rangeCheckForAdd(index);   //先判断插入位置是否超出范围
        modCount++;   //同上
        final int s;
        Object[] elementData;
      	//看着有点绕，但实际上就是让s等于size，elementData等于类的elementData
        //然后再比较长度是否已经一样，一样就扩容，跟上面思路是差不多的
        if ((s = size) == (elementData = this.elementData).length)
            elementData = grow();
        System.arraycopy(elementData, index,
                         elementData, index + 1,
                         s - index);  //因为是中间插入，这里调用C++实现的数组移动操作，把位置让出来
        elementData[index] = element;  //位置让出来之后，设置新元素
        size = s + 1;  //让size自增
    }
  	
  	...
  
    private Object[] grow() {
        return grow(size + 1);   //调用内部其他方法实现，并制定扩容最小值为当前容量+1
    }
  
  	private Object[] grow(int minCapacity) {
        int oldCapacity = elementData.length;   //首先保存下现在的容量
        if (oldCapacity > 0 || elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            //判断当前容量是否不为0（因为初始就是0，不为0一定被扩容过）
            //然后计算新的容量，这里传入当前长度、最小扩容长度和推荐扩容长度三个参数，通过辅助方法衡量该如何扩容
            int newCapacity = ArraysSupport.newLength(oldCapacity,
                    minCapacity - oldCapacity, /* 最小扩容长度 */
                    oldCapacity >> 1           /* 推荐扩容长度 */);
            return elementData = Arrays.copyOf(elementData, newCapacity);  //得到最终扩容大小，创建新数组
        } else {
            //这里相当于现在容量为0，也就是初始状态，此时会直接创建一个容量为10的新数组
            //注意这里需要取minCapacity和默认容量的最大值，因为grow不仅仅在单个插入时会调用，批量插入的时候也会调用，有可能出现批量插入20个的情况，那么初始容量就装不下了
            return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];
        }
    }
}
```

其中具体的ArrayList扩容辅助类定义如下：

```java
		//默认的列表最大长度为Integer.MAX_VALUE - 8
    //因为在部分JVM的C++实现中，在数组的对象头中有一个_length字段，用于记录数组的长度
    //所以这个8（保守估计，不一定是）就是存了数组_length字段（这个只做了解就行）
    public static final int SOFT_MAX_ARRAY_LENGTH = Integer.MAX_VALUE - 8;

    public static int newLength(int oldLength, int minGrowth, int prefGrowth) {
        //计算新的长度，因为minGrowth和prefGrowth谁更大不确定，需要进行比较，然后得到新的长度
        int prefLength = oldLength + Math.max(minGrowth, prefGrowth);
        //接着比较新的长度是否已经超出数组允许的最大长度了
        if (0 < prefLength && prefLength <= SOFT_MAX_ARRAY_LENGTH) {
            return prefLength;  //没有直接返回
        } else {
            // 如果超出最大长度，需要进一步处理
            return hugeLength(oldLength, minGrowth);
        }
    }

		private static int hugeLength(int oldLength, int minGrowth) {
        //先看看现在需要的最小长度，因为走到这里有可能是因为prefLength过长，但并不代表就真的需要这么长，因为prefLength有可能加的是prefGrowth，不一定是最小值
        int minLength = oldLength + minGrowth;  
        if (minLength < 0) { // 如果加出来最小长度已经小于0了，那包是超过int最大值了（前面二进制章节有介绍为什么）
            throw new OutOfMemoryError(   //直接无情抛异常
                "Required array length " + oldLength + " + " + minGrowth + " is too large");
        } else if (minLength <= SOFT_MAX_ARRAY_LENGTH) {  //如果在最大长度允许范围内
            return SOFT_MAX_ARRAY_LENGTH;   //直接给最大的
        } else {
            return minLength;  
            //这种情况相当于在SOFT_MAX_ARRAY_LENGTH和int最大值之间，没法了，只能直接返回
            //虽然有些JVM会可能直接抛出异常，但是可以抱着试试的心态搞一下
        }
    }
```

所以Java为我们提供的ArrayList，默认情况下内部就是一个空的数组，需要使用时候会变成初始值10（或初始批量插入的长度）后续在单个插入时，如果容量不够，会自动按照1.5倍进行扩容，直到最大限制。如果是后续批量插入，则根据情况而定。

一般的，如果我们要使用一个集合类，我们会使用接口的引用：

```java
public static void main(String[] args) {
    List<String> list = new ArrayList<>();   //使用接口的引用来操作具体的集合类实现，是为了方便日后如果我们想要更换不同的集合类实现，而且接口中本身就已经定义了主要的方法，所以说没必要直接用实现类
    list.add("科技与狠活");   //使用add添加元素
  	list.add("上头啊");
    System.out.println(list);   //打印集合类，可以得到一个非常规范的结果
}
```

可以看到，打印集合类的效果，跟我们使用Arrays工具类是一样的：

![image-20221001002151164](https://s2.loli.net/2022/10/01/v3uzfnhamXV5St8.png)

集合的各种功能我们都可以来测试一下，特别注意一下，我们在使用Integer时，要注意传参问题：

```java
public static void main(String[] args) {
    List<Integer> list = new ArrayList<>();
    list.add(10);   //添加Integer的值10
    list.remove((Integer) 10);   //注意，不能直接用10，默认情况下会认为传入的是int类型值，删除的是下标为10的元素，我们这里要删除的是刚刚传入的值为10的Integer对象
    System.out.println(list);   //可以看到，此时元素成功被移除
}
```

那要是这样写呢？

```java
public static void main(String[] args) {
    List<Integer> list = new ArrayList<>();
    list.add(new Integer(10));   //添加的是一个对象
    list.remove(new Integer(10));   //删除的是另一个对象
    System.out.println(list);
}
```

可以看到，结果依然是删除成功，这是因为集合类在删除元素时，只会调用`equals`方法进行判断是否为指定元素，而不是进行等号判断，所以说一定要注意，如果两个对象使用`equals`方法相等，那么集合中就是相同的两个对象：

```java
//ArrayList源码部分
public boolean remove(Object o) {
    if (o == null) {
        ...
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {   //这里只是对两个对象进行equals判断
                fastRemove(index);
                return true;  //只要判断成功，直接认为就是要删除的对象，删除就完事
            }
    }
    return false;
}
```

列表中允许存在相同元素，所以说我们可以添加两个一模一样的：

```java
public static void main(String[] args) {
    List<String> list = new ArrayList<>();
    String str = "哟唉嘛干你";
    list.add(str);
    list.add(str);
    System.out.println(list);
}
```

![image-20221001231509926](https://s2.loli.net/2022/10/01/paeKLsGntNVfHPT.png)

那要是此时我们删除对象呢，是一起删除还是只删除一个呢？

```java
public static void main(String[] args) {
    List<String> list = new ArrayList<>();
    String str = "哟唉嘛干你";
    list.add(str);
    list.add(str);
    list.remove(str);
    System.out.println(list);
}
```

![image-20221001231619391](https://s2.loli.net/2022/10/01/5HdFh74wlqbMoj6.png)

可以看到，这种情况下，只会删除排在前面的第一个元素。

集合类是支持嵌套使用的，一个集合中可以存放多个集合，套娃嘛，谁不会：

```java
public static void main(String[] args) {
    List<List<String>> list = new LinkedList<>();
    list.add(new LinkedList<>());   //集合中的每一个元素就是一个集合，这个套娃是可以一直套下去的
    System.out.println(list.get(0).isEmpty());
}
```

在Arrays工具类中，我们可以快速生成一个只读的List：

```java
public static void main(String[] args) {
    List<String> list = Arrays.asList("A", "B", "C");   //非常方便
    System.out.println(list);
}
```

注意，这个生成的List是只读的，不能进行修改操作，只能使用获取内容相关的方法，否则抛出 UnsupportedOperationException 异常。要生成正常使用的，我们可以将这个只读的列表作为参数传入：

```java
public static void main(String[] args) {
    List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
    System.out.println(list);
}
```

当然，也可以利用类中的代码块实现：

```java
public static void main(String[] args) {
    List<String> list = new ArrayList<String>() {{   //使用匿名内部类（匿名内部类在Java8无法使用钻石运算符，但是之后的版本可以）
            add("A");
            add("B");
            add("C");
    }};
    System.out.println(list);
}
```

这里我们接着介绍另一个列表实现类，LinkedList同样是List的实现类，只不过它是采用的链式实现，也就是我们之前讲解的链表，只不过它是一个双向链表，也就是同时保存两个方向：

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    transient int size = 0;

    //引用首结点
    transient Node<E> first;
    //引用尾结点
    transient Node<E> last;

    //构造方法，很简单，直接创建就行了
    public LinkedList() {
    }
  
  	...
      
    private static class Node<E> {   //内部使用的结点类
        E item;
        Node<E> next;   //不仅保存指向下一个结点的引用，还保存指向上一个结点的引用
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
  
    public boolean add(E e) {
        linkLast(e);   //调用内部方法进行尾部连接
        return true;
    }
  
  	void linkLast(E e) {
        final Node<E> l = last;   //拿到当前尾结点
        final Node<E> newNode = new Node<>(l, e, null);  //创建新结点
        last = newNode;   //更新尾结点
        if (l == null)  
            first = newNode;  //如果当前尾结点为空说明是个空集合，将此结点同时作为首结点
        else
            l.next = newNode;  //否则更新当前尾结点的next引用，把新的结点串起来
        size++;   //让size自增
        modCount++;
    }
  
    E unlink(Node<E> x) {
        final E element = x.item;  //拿到当前的待删除结点元素
        final Node<E> next = x.next;   //拿到前后结点
        final Node<E> prev = x.prev;

        if (prev == null) {
            first = next;   //如果前驱结点为空，说明这个结点就是就是第一个，直接让first等于next就行了
        } else {
            prev.next = next;  //否则让前驱结点直接去连接下一个
            x.prev = null;   //取消当前结点对于前驱结点的引用，便于JVM自动垃圾回收
        }

        if (next == null) {  //同上，处于后驱结点，操作差不多
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;   //前后处理干净，然后把待删除结点对于元素的引用取消，彻底废弃掉此结点
        size--;   //让size自减
        modCount++;
        return element;  //返回被删元素
    }
  
    ...
}
```

LinkedList的使用和ArrayList的使用几乎相同，各项操作的结果也是一样的，在什么使用使用ArrayList和LinkedList，我们需要结合具体的场景来决定，尽可能的扬长避短。

* **ArrayList：** 更适合随机访问，因为可以直接读取某个下标的元素。插入则性能较差，因为需要移动一组元素，让出空间。
* **LinkedList：** 不适合随机访问，因为无法直接获取某个元素，只能遍历查找。插入性能较好，因为可以直接改变链表中结点的指向。

如果更多的是对数据进行插入，选择LinkedList，如果更多是对于数据的查询，选择ArrayList。

只不过LinkedList不仅可以当做List来使用，也可以当做双端队列使用，我们会在后面进行详细介绍。

### 迭代器

我们接着来介绍迭代器，实际上我们的集合类都是支持使用`foreach`语法的：

```java
public static void main(String[] args) {
    List<String> list = Arrays.asList("A", "B", "C");
    for (String s : list) {   //集合类同样支持这种语法
        System.out.println(s);
    }
}
```

但是由于仅仅是语法糖，实际上编译之后：

```java
public static void main(String[] args) {
    List<String> list = Arrays.asList("A", "B", "C");
    Iterator var2 = list.iterator();   //这里使用的是List的迭代器在进行遍历操作

    while(var2.hasNext()) {
        String s = (String)var2.next();
        System.out.println(s);
    }

}
```

那么这个迭代器是一个什么东西呢？我们来研究一下：

```java
public static void main(String[] args) {
    List<String> list = Arrays.asList("A", "B", "C");
  	//通过调用iterator方法快速获取当前集合的迭代器
  	//Iterator迭代器本身也是一个接口，由具体的集合实现类来根据情况实现
    Iterator<String> iterator = list.iterator();
}
```

通过使用迭代器，我们就可以实现对集合中的元素的进行遍历，就像我们遍历数组那样，它的运作机制大概是：

![image-20221002150914323](https://s2.loli.net/2022/10/02/8KS5jbTv7LoAVOs.png)

一个新的迭代器就像上面这样，默认有一个指向集合中第一个元素的指针：

![image-20221002151110991](https://s2.loli.net/2022/10/02/HxjfipVB9TlEbz5.png)

每一次`next`操作，都会将指针后移一位，直到完成每一个元素的遍历，此时再调用`next`将不能再得到下一个元素。至于为什么要这样设计，是因为集合类的实现方案有很多，可能是链式存储，也有可能是数组存储，不同的实现有着不同的遍历方式，而迭代器则可以将多种多样不同的集合类遍历方式进行统一，只需要各个集合类根据自己的情况进行对应实现就行了。

我们来看看这个接口的源码定义了哪些操作：

```java
public interface Iterator<E> {
    //看看是否还有下一个元素
    boolean hasNext();

    //遍历当前元素，并将下一个元素作为待遍历元素
    E next();

    //移除上一个被遍历的元素（某些集合不支持这种操作）
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }

    //对剩下的元素进行自定义遍历操作
    default void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (hasNext())
            action.accept(next());
    }
}
```

在ArrayList和LinkedList中，迭代器的实现也不同，比如ArrayList就是直接按下标访问：

```java
public E next() {
    ...
    cursor = i + 1;   //移动指针
    return (E) elementData[lastRet = i];  //直接返回指针所指元素
}
```

LinkedList就是不断向后寻找结点：

```java
public E next() {
    ...
    next = next.next;   //向后继续寻找结点
    nextIndex++;
    return lastReturned.item;  //返回结点内部存放的元素
}
```

虽然这两种列表的实现不同，遍历方式也不同，但是都是按照迭代器的标准进行了实现，所以说，我们想要遍历一个集合中所有的元素，那么就可以直接使用迭代器来完成，而不需要关心集合类是如何实现，我们该怎么去遍历：

```java
public static void main(String[] args) {
    List<String> list = Arrays.asList("A", "B", "C");
    Iterator<String> iterator = list.iterator();
    while (iterator.hasNext()) {    //每次循环一定要判断是否还有元素剩余
        System.out.println(iterator.next());  //如果有就可以继续获取到下一个元素
    }
}
```

注意，迭代器的使用是一次性的，用了之后就不能用了，如果需要再次进行遍历操作，那么需要重新生成一个迭代器对象。为了简便，我们可以直接使用`foreach`语法来快速遍历集合类，效果是完全一样的：

```java
public static void main(String[] args) {
    List<String> list = Arrays.asList("A", "B", "C");
    for (String s : list) {
        System.out.println(s);
    }
}
```

在Java8提供了一个支持Lambda表达式的forEach方法，这个方法接受一个Consumer，也就是对遍历的每一个元素进行的操作：

```java
public static void main(String[] args) {
    List<String> list = Arrays.asList("A", "B", "C");
    list.forEach(System.out::println);
    //这里有一个常见误区，lambda里面禁止修改外部的非final变量
}
```

这个效果跟上面的写法是完全一样的，因为forEach方法内部本质上也是迭代器在处理，这个方法是在Iterable接口中定义的：

```java
default void forEach(Consumer<? super T> action) {
    Objects.requireNonNull(action);
    for (T t : this) {   //foreach语法遍历每一个元素
        action.accept(t);   //调用Consumer的accept来对每一个元素进行消费
    }
}
```

那么我们来看一下，Iterable这个接口又是是什么东西？

![image-20221002152713622](https://s2.loli.net/2022/10/02/4ShtiO6kdIcwZ85.png)

我们来看看定义了哪些内容：

```java
//注意这个接口是集合接口的父接口，不要跟之前的迭代器接口搞混了
public interface Iterable<T> {
    //生成当前集合的迭代器，在Collection接口中重复定义了一次
    Iterator<T> iterator();

    //Java8新增方法，因为是在顶层接口中定义的，因此所有的集合类都有这个方法
    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }

    //这个方法会在多线程部分中进行介绍，暂时不做讲解
    default Spliterator<T> spliterator() {
        return Spliterators.spliteratorUnknownSize(iterator(), 0);
    }
}
```

得益于Iterable提供的迭代器生成方法，实际上只要是实现了迭代器接口的类（我们自己写的都行），都可以使用`foreach`语法：

```java
public class Test implements Iterable<String>{   //这里我们随便写一个类，让其实现Iterable接口
    @Override
    public Iterator<String> iterator() {
        return new Iterator<String>() {   //生成一个匿名的Iterator对象
            @Override
            public boolean hasNext() {   //这里随便写的，直接返回true，这将会导致无限循环
                return true;
            }

            @Override
            public String next() {   //每次就直接返回一个字符串吧
                return "测试";
            }
        };
    }
}
```

可以看到，直接就支持这种语法了，虽然我们这个是自己写的，并不是集合类：

```java
public static void main(String[] args) {
    Test test = new Test();
    for (String s : test) {
        System.out.println(s);
    }
}
```

![image-20221002154018319](https://s2.loli.net/2022/10/02/KejcFB8TChE5z4o.png)

是不是感觉集合类的设计非常巧妙？

我们这里再来介绍一下ListIterator，这个迭代器是针对于List的强化版本，增加了更多方便的操作，因为List是有序集合，所以它支持两种方向的遍历操作，不仅能从前向后，也可以从后向前：

```java
public interface ListIterator<E> extends Iterator<E> {
    //原本就有的
    boolean hasNext();

    //原本就有的
    E next();

    //查看前面是否有已经遍历的元素
    boolean hasPrevious();

    //跟next相反，这里是倒着往回遍历
    E previous();

    //返回下一个待遍历元素的下标
    int nextIndex();

    //返回上一个已遍历元素的下标
    int previousIndex();

    //原本就有的
    void remove();

    //将上一个已遍历元素修改为新的元素
    void set(E e);

    //在遍历过程中，插入新的元素到当前待遍历元素之前
    void add(E e);
}
```

我们来测试一下吧：

```java
public static void main(String[] args) {
    List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
    ListIterator<String> iterator = list.listIterator();
    iterator.next();   //此时得到A
    iterator.set("X");  //将A原本位置的上的元素设定为成新的
    System.out.println(list);
}
```

![image-20221002154844743](https://s2.loli.net/2022/10/02/C3xNDTEWGaPLfO6.png)

这种迭代器因为能够双向遍历，所以说可以反复使用。

### Queue和Deque

通过前面的学习，我们已经了解了List的使用，其中LinkedList除了可以直接当做列表使用之外，还可以当做其他的数据结构使用，可以看到它不仅仅实现了List接口：

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
```

这个Deque接口是干嘛的呢？我们先来看看它的继承结构：

![image-20221002162108279](https://s2.loli.net/2022/10/02/sCMgv9rl5b743BE.png)

我们先来看看队列接口，它扩展了大量队列相关操作：

```java
public interface Queue<E> extends Collection<E> {
    //队列的添加操作，是在队尾进行插入（只不过List也是一样的，默认都是尾插）
  	//如果插入失败，会直接抛出异常
    boolean add(E e);

    //同样是添加操作，但是插入失败不会抛出异常
    boolean offer(E e);

    //移除队首元素，但是如果队列已经为空，那么会抛出异常
    E remove();

   	//同样是移除队首元素，但是如果队列为空，会返回null
    E poll();

    //仅获取队首元素，不进行出队操作，但是如果队列已经为空，那么会抛出异常
    E element();

    //同样是仅获取队首元素，但是如果队列为空，会返回null
    E peek();
}
```

我们可以直接将一个LinkedList当做一个队列来使用：

```java
public static void main(String[] args) {
    Queue<String> queue = new LinkedList<>();   //当做队列使用，还是很方便的
    queue.offer("AAA");
    queue.offer("BBB");
    System.out.println(queue.poll());
    System.out.println(queue.poll());
}
```

![image-20221002163512442](https://s2.loli.net/2022/10/02/veHxlUkKyVYErgm.png)

我们接着来看双端队列，实际上双端队列就是队列的升级版，我们一个普通的队列就是：

![image-20220725103600318](https://s2.loli.net/2022/07/25/xBuZckTNtR54AEq.png)

普通队列中从队尾入队，队首出队，而双端队列允许在队列的两端进行入队和出队操作：

![image-20221002164302507](https://s2.loli.net/2022/10/02/gn8i3teclAKbhQS.png)

![image-20221002164431746](https://s2.loli.net/2022/10/02/in8IX3QkwtsLgWN.png)

利用这种特性，双端队列既可以当做普通队列使用，也可以当做栈来使用，我们来看看Java中是如何定义的Deque双端队列接口的：

```java
//在双端队列中，所有的操作都有分别对应队首和队尾的
public interface Deque<E> extends Queue<E> {
    //在队首进行插入操作
    void addFirst(E e);

    //在队尾进行插入操作
    void addLast(E e);
		
  	//不用多说了吧？
    boolean offerFirst(E e);
    boolean offerLast(E e);

    //在队首进行移除操作
    E removeFirst();

    //在队尾进行移除操作
    E removeLast();

    //不用多说了吧？
    E pollFirst();
    E pollLast();

    //获取队首元素
    E getFirst();

    //获取队尾元素
    E getLast();

		//不用多说了吧？
    E peekFirst();
    E peekLast();

    //从队列中删除第一个出现的指定元素
    boolean removeFirstOccurrence(Object o);

    //从队列中删除最后一个出现的指定元素
    boolean removeLastOccurrence(Object o);

    // *** 队列中继承下来的方法操作是一样的，这里就不列出了 ***

    ...

    // *** 栈相关操作已经帮助我们定义好了 ***

    //将元素推向栈顶
    void push(E e);

    //将元素从栈顶出栈
    E pop();


    // *** 集合类中继承的方法这里也不多种介绍了 ***

    ...

    //生成反向迭代器，这个迭代器也是单向的，但是是next方法是从后往前进行遍历的
    Iterator<E> descendingIterator();

}
```

我们可以来测试一下，比如我们可以直接当做栈来进行使用：

```java
public static void main(String[] args) {
    Deque<String> deque = new LinkedList<>();
    deque.push("AAA");
    deque.push("BBB");
    System.out.println(deque.pop());
    System.out.println(deque.pop());
}
```

![image-20221002165618791](https://s2.loli.net/2022/10/02/92woGL5MiBsTcKe.png)

可以看到，得到的顺序和插入顺序是完全相反的，其实只要各位理解了前面讲解的数据结构，就很简单了。我们来测试一下反向迭代器和正向迭代器：

```java
public static void main(String[] args) {
    Deque<String> deque = new LinkedList<>();
    deque.addLast("AAA");
    deque.addLast("BBB");
    
    Iterator<String> descendingIterator = deque.descendingIterator();
    System.out.println(descendingIterator.next());

    Iterator<String> iterator = deque.iterator();
    System.out.println(iterator.next());
}
```

可以看到，正向迭代器和反向迭代器的方向是完全相反的。

当然，除了LinkedList实现了队列接口之外，还有其他的实现类，但是并不是很常用，这里做了解就行了：

```java
public static void main(String[] args) {
    Deque<String> deque = new ArrayDeque<>();   //数组实现的栈和队列
    Queue<String> queue = new PriorityQueue<>();  //优先级队列
}
```

这里需要介绍一下优先级队列，优先级队列可以根据每一个元素的优先级，对出队顺序进行调整，默认情况按照自然顺序：

```java
public static void main(String[] args) {
    Queue<Integer> queue = new PriorityQueue<>();
    queue.offer(10);
    queue.offer(4);
    queue.offer(5);
    System.out.println(queue.poll());
    System.out.println(queue.poll());
    System.out.println(queue.poll());
}
```

![image-20221003210253093](https://s2.loli.net/2022/10/03/bmEP9fgCS1Ksaqw.png)

可以看到，我们的插入顺序虽然是10/4/5，但是出队顺序是按照优先级来的，类似于VIP用户可以优先结束排队。我们也可以自定义比较规则，同样需要给一个Comparator的实现：

```java
public static void main(String[] args) {
    Queue<Integer> queue = new PriorityQueue<>((a, b) -> b - a);   //按照从大到小顺序出队
    queue.offer(10);
    queue.offer(4);
    queue.offer(5);
    System.out.println(queue.poll());
    System.out.println(queue.poll());
    System.out.println(queue.poll());
}
```

![image-20221003210436684](https://s2.loli.net/2022/10/03/G5SZgKxvUJyPABD.png)

只不过需要注意的是，优先级队列并不是队列中所有的元素都是按照优先级排放的，优先级队列**只能保证出队顺序是按照优先级**进行的，我们可以打印一下：

![image-20221003210545678](https://s2.loli.net/2022/10/03/9dSheG4xqFoXB5i.png)

想要了解优先级队列的具体实现原理，可以在《数据结构与算法》篇视频教程中学习大顶堆和小顶堆。

### Set集合

前面我们已经介绍了列表，我们接着来看Set集合，这种集合类型比较特殊，我们先来看看Set的定义：

```java
public interface Set<E> extends Collection<E> {
    // Set集合中基本都是从Collection直接继承过来的方法，只不过对这些方法有更加特殊的定义
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    Iterator<E> iterator();
    Object[] toArray();
    <T> T[] toArray(T[] a);

    //添加元素只有在当前Set集合中不存在此元素时才会成功，如果插入重复元素，那么会失败
    boolean add(E e);

    //这个同样是删除指定元素
    boolean remove(Object o);

    boolean containsAll(Collection<?> c);

    //同样是只能插入那些不重复的元素
    boolean addAll(Collection<? extends E> c);
  
    boolean retainAll(Collection<?> c);
    boolean removeAll(Collection<?> c);
    void clear();
    boolean equals(Object o);
    int hashCode();

    //这个方法我们同样会放到多线程中进行介绍
    @Override
    default Spliterator<E> spliterator() {
        return Spliterators.spliterator(this, Spliterator.DISTINCT);
    }
}
```

我们发现接口中定义的方法都是Collection中直接继承的，因此，Set支持的功能其实也就和Collection中定义的差不多，只不过：

- 不允许出现重复元素
- 不支持随机访问（不允许通过下标访问）

首先认识一下HashSet，它的底层就是采用哈希表实现的（我们在这里先不去探讨实现原理，因为底层实质上是借用的一个HashMap在实现，这个需要我们学习了Map之后再来讨论）我们可以非常高效的从HashSet中存取元素，我们先来测试一下它的特性：

```java
public static void main(String[] args) {
    Set<String> set = new HashSet<>();
    System.out.println(set.add("AAA"));   //这里我们连续插入两个同样的字符串
    System.out.println(set.add("AAA"));
    System.out.println(set);   //可以看到，最后实际上只有一个成功插入了
}
```

![image-20221003211330129](https://s2.loli.net/2022/10/03/y5AoUG1iuWzhOSj.png)

在Set接口中并没有定义支持指定下标位置访问的添加和删除操作，我们只能简单的删除Set中的某个对象：

```java
public static void main(String[] args) {
    Set<String> set = new HashSet<>();
    System.out.println(set.add("AAA"));
    System.out.println(set.remove("AAA"));
    System.out.println(set);
}
```

由于底层采用哈希表实现，所以说无法维持插入元素的顺序：

```java
public static void main(String[] args) {
    Set<String> set = new HashSet<>();
    set.addAll(Arrays.asList("A", "0", "-", "+"));
    System.out.println(set);
}
```

![image-20221003211635759](https://s2.loli.net/2022/10/03/OekDqMlpVbxImsK.png)

那要是我们就是想要使用维持顺序的Set集合呢？我们可以使用LinkedHashSet，LinkedHashSet底层维护的不再是一个HashMap，而是LinkedHashMap，它能够在插入数据时利用链表自动维护顺序，因此这样就能够保证我们插入顺序和最后的迭代顺序一致了。

```java
public static void main(String[] args) {
    Set<String> set = new LinkedHashSet<>();
    set.addAll(Arrays.asList("A", "0", "-", "+"));
    System.out.println(set);
}
```

![image-20221003212147700](https://s2.loli.net/2022/10/03/TpczL2Zi1OkaHWI.png)

还有一种Set叫做TreeSet，它会在元素插入时进行排序：

```java
public static void main(String[] args) {
    TreeSet<Integer> set = new TreeSet<>();
    set.add(1);
    set.add(3);
    set.add(2);
    System.out.println(set);
}
```

![image-20221003212233263](https://s2.loli.net/2022/10/03/3VwDQzRxUTGrOZb.png)

可以看到最后得到的结果并不是我们插入顺序，而是按照数字的大小进行排列。当然，我们也可以自定义排序规则：

```java
public static void main(String[] args) {
    TreeSet<Integer> set = new TreeSet<>((a, b) -> b - a);  //同样是一个Comparator
    set.add(1);
    set.add(3);
    set.add(2);
    System.out.println(set);
}
```

目前，Set集合只是粗略的进行了讲解，但是学习Map之后，我们还会回来看我们Set的底层实现，所以说最重要的还是Map。本节只需要记住Set的性质、使用即可。

### Map映射

什么是映射？我们在高中阶段其实已经学习过映射（Mapping）了，映射指两个元素的之间相互“对应”的关系，也就是说，我们的元素之间是两两对应的，是以键值对的形式存在。

![39e19f3e-04e8-4c43-8fb5-6d5288a7cdf8](https://s2.loli.net/2022/10/03/QSxqJLwiNM1nZlO.jpg)

而Map就是为了实现这种数据结构而存在的，我们通过保存键值对的形式来存储映射关系，就可以轻松地通过键找到对应的映射值，比如现在我们要保存很多学生的信息，而这些学生都有自己的ID，我们可以将其以映射的形式保存，将ID作为键，学生详细信息作为值，这样我们就可以通过学生的ID快速找到对应学生的信息了。

![image-20221003213157956](https://s2.loli.net/2022/10/03/i2x6m3hzFC5GIAd.png)

在Map中，这些映射关系被存储为键值对，我们先来看看Map接口中定义了哪些操作：

```java
//Map并不是Collection体系下的接口，而是单独的一个体系，因为操作特殊
//这里需要填写两个泛型参数，其中K就是键的类型，V就是值的类型，比如上面的学生信息，ID一般是int，那么键就是Integer类型的，而值就是学生信息，所以说值是学生对象类型的
public interface Map<K,V> {
    //-------- 查询相关操作 --------
  
  	//获取当前存储的键值对数量
    int size();

    //是否为空
    boolean isEmpty();

    //查看Map中是否包含指定的键
    boolean containsKey(Object key);

    //查看Map中是否包含指定的值
    boolean containsValue(Object value);

    //通过给定的键，返回其映射的值
    V get(Object key);

    //-------- 修改相关操作 --------

    //向Map中添加新的映射关系，也就是新的键值对
    V put(K key, V value);

    //根据给定的键，移除其映射关系，也就是移除对应的键值对
    V remove(Object key);


    //-------- 批量操作 --------

    //将另一个Map中的所有键值对添加到当前Map中
    void putAll(Map<? extends K, ? extends V> m);

    //清空整个Map
    void clear();


    //-------- 其他视图操作 --------

    //返回Map中存放的所有键，以Set形式返回
    Set<K> keySet();

    //返回Map中存放的所有值
    Collection<V> values();

    //返回所有的键值对，这里用的是内部类Entry在表示
    Set<Map.Entry<K, V>> entrySet();

    //这个是内部接口Entry，表示一个键值对
    interface Entry<K,V> {
        //获取键值对的键
        K getKey();

        //获取键值对的值
        V getValue();

        //修改键值对的值
        V setValue(V value);

        //判断两个键值对是否相等
        boolean equals(Object o);

        //返回当前键值对的哈希值
        int hashCode();

        ...
    }

    ...
}
```

当然，Map中定义了非常多的方法，尤其是在Java 8之后新增的大量方法，我们会在后面逐步介绍的。

我们可以来尝试使用一下Map，实际上非常简单，这里我们使用最常见的HashMap，它的底层采用哈希表实现：

```java
public static void main(String[] args) {
    Map<Integer, String> map = new HashMap<>();
    map.put(1, "小明");   //使用put方法添加键值对，返回值我们会在后面讨论
    map.put(2, "小红");
    System.out.println(map.get(2)); //使用get方法根据键获取对应的值
}
```

![image-20221003214807048](https://s2.loli.net/2022/10/03/8Fl6YizINQP9dmX.png)

注意，Map中无法添加相同的键，同样的键只能存在一个，即使值不同。如果出现键相同的情况，那么会覆盖掉之前的：

```java
public static void main(String[] args) {
    Map<Integer, String> map = new HashMap<>();
    map.put(1, "小明");
    map.put(1, "小红");   //这里的键跟之前的是一样的，这样会导致将之前的键值对覆盖掉
    System.out.println(map.get(1));
}
```

![image-20221003214807048](https://s2.loli.net/2022/10/03/8Fl6YizINQP9dmX.png)

为了防止意外将之前的键值对覆盖掉，我们可以使用：

```java
public static void main(String[] args) {
    Map<Integer, String> map = new HashMap<>();
    map.put(1, "小明");
    map.putIfAbsent(1, "小红");   //Java8新增操作，只有在不存在相同键的键值对时才会存放
    System.out.println(map.get(1));
}
```

还有，我们在获取一个不存在的映射时，默认会返回null作为结果：

```java
public static void main(String[] args) {
    Map<Integer, String> map = new HashMap<>();
    map.put(1, "小明");   //Map中只有键为1的映射
    System.out.println(map.get(3));  //此时获取键为3的值，那肯定是没有的，所以说返回null
}
```

我们也可以为这种情况添加一个预备方案，当Map中不存在时，可以返回一个备选的返回值：

```java
public static void main(String[] args) {
    Map<Integer, String> map = new HashMap<>();
    map.put(1, "小明");
    System.out.println(map.getOrDefault(3, "备胎"));   //Java8新增操作，当不存在对应的键值对时，返回备选方案
}
```

同样的，因为HashMap底层采用哈希表实现，所以不维护顺序，我们在获取所有键和所有值时，可能会是乱序的：

```java
public static void main(String[] args) {
    Map<String , String> map = new HashMap<>();
    map.put("0", "十七张");
    map.put("+", "牌");
    map.put("P", "你能秒我");
    System.out.println(map);
    System.out.println(map.keySet());
    System.out.println(map.values());
}
```

![image-20221003220156062](https://s2.loli.net/2022/10/03/DNXqwk3UOPnMmlc.png)

如果需要维护顺序，我们同样可以使用LinkedHashMap，它的内部对插入顺序进行了维护：

```java
public static void main(String[] args) {
    Map<String , String> map = new LinkedHashMap<>();
    map.put("0", "十七张");
    map.put("+", "牌");
    map.put("P", "你能秒我");
    System.out.println(map);
    System.out.println(map.keySet());
    System.out.println(map.values());
}
```

![image-20221003220458539](https://s2.loli.net/2022/10/03/QHkWZsFvzASpxqL.png)

实际上Map的使用还是挺简单的，我们接着来看看Map的底层是如何实现的，首先是最简单的HashMap，我们前面已经说过了，它的底层采用的是哈希表，首先回顾我们之前学习的哈希表，我们当时说了，哈希表可能会出现哈希冲突，这样保存的元素数量就会存在限制，而我们可以通过链地址法解决这种问题，最后哈希表就长这样了：

![image-20220820221104298](https://s2.loli.net/2022/09/30/kr4CcVEwI72AiDU.png)

实际上这个表就是一个存放头结点的数组+若干结点，而HashMap也是差不多的实现，我们来看看这里面是怎么定义的：

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
  
  	...
    
  	static class Node<K,V> implements Map.Entry<K,V> {   //内部使用结点，实际上就是存放的映射关系
        final int hash;
        final K key;   //跟我们之前不一样，我们之前一个结点只有键，而这里的结点既存放键也存放值，当然计算哈希还是使用键
        V value;
        Node<K,V> next;
				...
    }
  	
  	...
  
  	transient Node<K,V>[] table;   //这个就是哈希表本体了，可以看到跟我们之前的写法是一样的，也是结点数组，只不过HashMap中没有设计头结点（相当于没有头结点的链表）
  
  	final float loadFactor;   //负载因子，这个东西决定了HashMap的扩容效果
  
  	public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; //当我们创建对象时，会使用默认的负载因子，值为0.75
    }
  
  	...     
}
```

可以看到，实际上底层大致结构跟我们之前学习的差不多，只不过多了一些特殊的东西：

* HashMap支持自动扩容，哈希表的大小并不是一直不变的，否则太过死板
* HashMap并不是只使用简单的链地址法，当链表长度到达一定限制时，会转变为效率更高的红黑树结构

我们来研究一下它的put方法：

```java
public V put(K key, V value) {
  	//这里计算完键的哈希值之后，调用的另一个方法进行映射关系存放
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)  //如果底层哈希表没初始化，先初始化
        n = (tab = resize()).length;   //通过resize方法初始化底层哈希表，初始容量为16，后续会根据情况扩容，底层哈希表的长度永远是2的n次方
  	//因为传入的哈希值可能会很大，这里同样是进行取余操作
  	//这里做了个优化 (n - 1) & hash 等价于 hash % n （仅限n为2的幂可以这样做）这里的i就是最终得到的下标位置了
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = new Node(hash, key, value, null);   //如果这个位置上什么都没有，那就直接放一个新的结点
    else {   //这种情况就是哈希冲突了
        Node<K,V> e; K k;
        if (p.hash == hash &&   //如果上来第一个结点的键的哈希值跟当前插入的键的哈希值相同，键也相同，说明已经存放了相同键的键值对了，那就执行覆盖操作
            ((k = p.key) == key || (key != null && key.equals(k))))  //依然是equals判断
            e = p;   //这里直接将待插入结点等于原本冲突的结点，一会直接覆盖
        else if (p instanceof TreeNode)   //如果第一个结点是TreeNode类型的，说明这个链表已经升级为红黑树了
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);  //在红黑树中插入新的结点
        else {
            for (int binCount = 0; ; ++binCount) {  //普通链表就直接在链表尾部插入
                if ((e = p.next) == null) {   //e每次都更新为p.next
                    p.next = newNode(hash, key, value, null);  //找到尾部，直接创建新的结点连在后面
                    if (binCount >= TREEIFY_THRESHOLD - 1) //如果当前链表的长度已经很长了，达到了阈值
                        treeifyBin(tab, hash);			//那么就转换为红黑树来存放
                    break;   //直接结束
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))  //同样的，如果在向下找的过程中发现已经存在相同键的键值对了，直接结束
                    break;
                p = e;
            }
        }
        if (e != null) { // 如果e不为空，只有可能是前面出现了相同键的情况，其他情况e都是null，所以直接覆盖就行
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;  //覆盖原本存储的值
            afterNodeAccess(e);
            return oldValue;   //覆盖之后，会返回原本的被覆盖值
        }
    }
    ++modCount;
    if (++size > threshold)   //键值对size计数自增，如果超过阈值，会对底层哈希表数组进行扩容
        resize();   //调用resize进行扩容
    afterNodeInsertion(evict);
    return null;  //正常插入键值对返回值为null
}
```

是不是感觉只要前面的数据结构听懂了，这里简直太简单。根据上面的推导，我们在正常插入一个键值对时，会得到null返回值，而冲突时会得到一个被覆盖的值：

```java
public static void main(String[] args) {
    Map<String , String> map = new HashMap<>();
    System.out.println(map.put("0", "十七张"));
    System.out.println(map.put("0", "慈善家"));
}
```

![image-20221003224137177](https://s2.loli.net/2022/10/03/A2rXocbU9StlDOC.png)

现在我们知道，当HashMap的一个链表长度过大时，会自动转换为红黑树：

![710c1c38-95a8-493d-8645-067b991af908](https://s2.loli.net/2022/10/03/E7GnIVjPAwf8Fol.jpg)

但是这样始终治标不治本，受限制的始终是底层哈希表的长度，我们还需要进一步对底层的这个哈希表进行扩容才可以从根本上解决问题，我们来看看`resize()`方法：

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;   //先把下面这几个旧的东西保存一下
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;  //这些是新的容量和扩容阈值
    if (oldCap > 0) {  //如果旧容量大于0，那么就开始扩容
        if (oldCap >= MAXIMUM_CAPACITY) {  //如果旧的容量已经大于最大限制了，那么直接给到 Integer.MAX_VALUE
            threshold = Integer.MAX_VALUE;
            return oldTab;  //这种情况不用扩了
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)   //新的容量等于旧容量的2倍，同样不能超过最大值
            newThr = oldThr << 1; //新的阈值也提升到原来的两倍
    }
    else if (oldThr > 0) // 旧容量不大于0只可能是还没初始化，这个时候如果阈值大于0，直接将新的容量变成旧的阈值
        newCap = oldThr;
    else {               // 默认情况下阈值也是0，也就是我们刚刚无参new出来的时候
        newCap = DEFAULT_INITIAL_CAPACITY;   //新的容量直接等于默认容量16
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY); //阈值为负载因子乘以默认容量，负载因子默认为0.75，也就是说只要整个哈希表用了75%的容量，那么就进行扩容，至于为什么默认是0.75，原因很多，这里就不解释了，反正作为新手，这些都是大佬写出来的，我们用就完事。
    }
    ...
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;   //将底层数组变成新的扩容之后的数组
    if (oldTab != null) {  //如果旧的数组不为空，那么还需要将旧的数组中所有元素全部搬到新的里面去
      	...   //详细过程就不介绍了
    }
}
```

是不是感觉自己有点了解HashMap的运作机制了，其实并不是想象中的那么难，因为这些东西再怎么都是人写的。

>  HashMap初始容量为0，第一次put的时候，会立即进行一次resize，第一次resize会直接生成一个数组作为哈希表，容量为16，阈值为16 * 0.75，当后续插入键值对超过阈值时，会进行第二次resize，第二次resize会将容量和阈值都变为原来的2倍，第三次以此类推。

而LinkedHashMap是直接继承自HashMap，具有HashMap的全部性质，同时得益于每一个节点都是一个双向链表，在插入键值对时，同时保存了插入顺序：

```java
static class Entry<K,V> extends HashMap.Node<K,V> {   //LinkedHashMap中的结点实现
    Entry<K,V> before, after;   //这里多了一个指向前一个结点和后一个结点的引用
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

这样我们在遍历LinkedHashMap时，顺序就同我们的插入顺序一致。

当然还有一种比较特殊的Map叫做TreeMap，就像它的名字一样，就是一个Tree，它的内部直接维护了一个红黑树（没有使用哈希表）因为它会将我们插入的结点按照规则进行排序，所以说直接采用红黑树会更好，我们在创建时，只需给予一个比较规则即可，跟之前的TreeSet是一样的：

```java
public static void main(String[] args) {
    Map<Integer , String> map = new TreeMap<>((a, b) -> b - a);
    map.put(0, "单走");
    map.put(1, "一个六");
    map.put(3, "**");
    System.out.println(map);
}
```

![image-20221003231135805](https://s2.loli.net/2022/10/03/2oJXBui5aD8q1Gh.png)

现在我们倒回来看之前讲解的HashSet集合，实际上它的底层很简单：

```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
{

    private transient HashMap<E,Object> map;   //对，你没看错，底层直接用map来做事

    // 因为Set只需要存储Key就行了，所以说这个对象当做每一个键值对的共享Value
    private static final Object PRESENT = new Object();

    //直接构造一个默认大小为16负载因子0.75的HashMap
    public HashSet() {
        map = new HashMap<>();
    }
		
  	...
      
    //你会发现所有的方法全是替身攻击
    public Iterator<E> iterator() {
        return map.keySet().iterator();
    }

    public int size() {
        return map.size();
    }

    public boolean isEmpty() {
        return map.isEmpty();
    }
}
```

通过观察HashSet的源码发现，HashSet几乎都在操作内部维护的一个HashMap，也就是说，HashSet只是一个表壳，而内部维护的HashMap才是灵魂！就像你进了公司，在外面花钱请别人帮你写公司的业务，你只需要坐着等别人写好然后你自己拿去交差就行了。所以说，HashSet利用了HashMap内部的数据结构，轻松地就实现了Set定义的全部功能！

再来看TreeSet，实际上用的就是我们的TreeMap：

```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable
{
    //底层需要一个NavigableMap，就是自动排序的Map
    private transient NavigableMap<E,Object> m;

    //不用我说了吧
    private static final Object PRESENT = new Object();

    ...

    //直接使用TreeMap解决问题
    public TreeSet() {
        this(new TreeMap<E,Object>());
    }
		
  	...
}
```

同理，这里就不多做阐述了。

我们接着来看看Map中定义的哪些杂七杂八的方法，首先来看看`compute`方法：

```java
public static void main(String[] args) {
    Map<Integer, String> map = new HashMap<>();
    map.put(1, "A");
    map.put(2, "B");
    map.compute(1, (k, v) -> {   //compute会将指定Key的值进行重新计算，若Key不存在，v会返回null
        return v+"M";     //这里返回原来的value+M
    });
  	map.computeIfPresent(1, (k, v) -> {   //当Key存在时存在则计算并赋予新的值
      return v+"M";     //这里返回原来的value+M
    });
    System.out.println(map);
}
```

也可以使用`computeIfAbsent`，当不存在Key时，计算并将键值对放入Map中：

```java
public static void main(String[] args) {
    Map<Integer, String> map = new HashMap<>();
    map.put(1, "A");
    map.put(2, "B");
    map.computeIfAbsent(0, (k) -> {   //若不存在则计算并插入新的值
        return "M";     //这里返回M
    });
    System.out.println(map);
}
```

merge方法用于处理数据：

```java
public static void main(String[] args) {
    List<Student> students = Arrays.asList(
            new Student("yoni", "English", 80),
            new Student("yoni", "Chiness", 98),
            new Student("yoni", "Math", 95),
            new Student("taohai.wang", "English", 50),
            new Student("taohai.wang", "Chiness", 72),
            new Student("taohai.wang", "Math", 41),
            new Student("Seely", "English", 88),
            new Student("Seely", "Chiness", 89),
            new Student("Seely", "Math", 92)
    );
    Map<String, Integer> scoreMap = new HashMap<>();
  	//merge方法可以对重复键的值进行特殊操作，比如我们想计算某个学生的所有科目分数之后，那么就可以像这样：
    students.forEach(student -> scoreMap.merge(student.getName(), student.getScore(), Integer::sum));
    scoreMap.forEach((k, v) -> System.out.println("key:" + k + "总分" + "value:" + v));
}

static class Student {
    private final String name;
    private final String type;
    private final int score;

    public Student(String name, String type, int score) {
        this.name = name;
        this.type = type;
        this.score = score;
    }

    public String getName() {
        return name;
    }

    public int getScore() {
        return score;
    }

    public String getType() {
        return type;
    }
}
```

`replace`方法可以快速替换某个映射的值：

```java
public static void main(String[] args) {
    Map<Integer , String> map = new HashMap<>();
    map.put(0, "单走");
    map.replace(0, ">>>");   //直接替换为新的
    System.out.println(map);
}
```

也可以精准匹配：

```java
public static void main(String[] args) {
    Map<Integer , String> map = new HashMap<>();
    map.put(0, "单走");
    map.replace(0, "巴卡", "玛卡");   //只有键和值都匹配时，才进行替换
    System.out.println(map);
}
```

包括remove方法，也支持键值同时匹配：

```java
public static void main(String[] args) {
    Map<Integer , String> map = new HashMap<>();
    map.put(0, "单走");
    map.remove(0, "单走");  //只有同时匹配时才移除
    System.out.println(map);
}
```

是不是感觉学习了Map之后，涨了不少姿势？

### 比较相关接口

在前面的学习中，我们发现很多的类中都存在一个`compareTo`方法，这实际上是因为这个类实现了`Comparable`接口获得的方法，我们常见的一些JDK提供的类型，如String、基本类型包装类、Date、LocalDate等，其实都实现了此接口，从而自带对于其类型对象之间的比较操作：

```java
String s1 = "AAA", s2 = "BBB";
System.out.println(s1.compareTo(s2));  //按字典顺序比较两个字符串,比较基于字符串中每个字符的 Unicode 值。
```

这与我们前面提到过的`Comparator`类似，当返回值大于0说明前者大于后者，小于0则是后者大于前者，如果等于0则表示这两者相等。但是注意`Comparator`是用于直接手动创建比较规则，而`Comparable`则是直接让类自行定义内部比较规则，包括这两者的使用的地方也不一样。

比如我们前面介绍的`Arrays`工具类，其中就包含`sort`方法进行排序，我们可以直接传入一个数组让其自动完成排序操作：

```java
String[] arr = { "DDD", "BBB", "CCC", "AAA" };
Arrays.sort(arr);
System.out.println(Arrays.toString(arr));
```

但是注意，我们进行排序的类型必须是一个实现了`Comparable`接口的类型，否则无法进行排序：

```java
public static void main(String[] args) {
    Student[] arr = { new Student(), new Student() };
    Arrays.sort(arr);
    System.out.println(Arrays.toString(arr));
}
    
static class Student {}
```

![image-20250722143249240](https://s2.loli.net/2025/07/22/vC7VGpdl268LWAk.png)

我们可以为我们自己定义的类实现`Comparable`接口，实现一个自己的比较操作即可：

```java
public static void main(String[] args) {
    Student[] arr = { new Student(19), new Student(18) };
    Arrays.sort(arr);
    System.out.println(Arrays.toString(arr));
}

static class Student implements Comparable<Student> {
    int age;

    public Student(int age) {
        this.age = age;
    }

    @Override
    public int compareTo(Student o) {
        return age - o.age;   //直接比较年龄
    }

    @Override
    public String toString() {
        return "Student{" +
               "age=" + age +
               '}';
    }
}
```

实际上我们之前介绍的`TreeSet`和`TreeMap`，在我们不指定任何排序规则的情况下，默认也是采用类中Comparable的实现进行排序的，如果我们直接使用一个为实现`Comparable`接口的类型，同样会出现报错：

![image-20250722145029512](https://s2.loli.net/2025/07/22/o4xXJcsAh7DUinL.png)

不过，除了让类本身实现`Comparable`接口之外，对于一些第三方提供的类型我们无法进行修改的，此时也可以手动指定比较规则，也就是通过传入一个`Comparator`的实现：

```java
public static void main(String[] args) {
    TreeSet<Student> set = new TreeSet<>((s1, s2) -> s1.age - s2.age);  //手动指定一个比较规则也可以
    set.add(new Student(19));
    set.add(new Student(18));
    System.out.println(set);
}

static class Student {
    int age;

    public Student(int age) {
        this.age = age;
    }
}
```

同样的，数组也有类似的`sort`方法，这里就不列出了。接下来我们就着重介绍一下`Comparator`接口：

```java
@FunctionalInterface
public interface Comparator<T> {
    //我们需要实现的比较方法
    int compare(T o1, T o2);

    //从Object继承下来的方法，这里进行了重新定义，
    //除了比较两个元素相等之外，它还额外要求两个元素compare得到的结果为0
    boolean equals(Object obj);

    //根据当前实现的compare操作，生成一个相反的比较Comparator
    default Comparator<T> reversed() {
        return Collections.reverseOrder(this);
    }

    //在原有比较基础上，附加次级比较条件（也就是说如果比较出来两个对象相等，就继续按照次级条件比较得到谁前谁后）
    default Comparator<T> thenComparing(Comparator<? super T> other) {
        Objects.requireNonNull(other);
        return (Comparator<T> & Serializable) (c1, c2) -> {
            int res = compare(c1, c2);
            return (res != 0) ? res : other.compare(c1, c2);
        };
    }

    //上面的进阶版，可以自定义类中属性(作为key)的获取规则，并对此属性进行比较
    default <U> Comparator<T> thenComparing(
            Function<? super T, ? extends U> keyExtractor,
            Comparator<? super U> keyComparator)
    {
        return thenComparing(comparing(keyExtractor, keyComparator));
    }

    //上面的简化版，只有类中属性(作为key)的获取规则，比较规则使用key默认的实现（Key必须是Comparable的实现类）
    default <U extends Comparable<? super U>> Comparator<T> thenComparing(
            Function<? super T, ? extends U> keyExtractor)
    {
        return thenComparing(comparing(keyExtractor));
    }

    //针对于特定内置类型的提取并比较，同上，后续同理
    default Comparator<T> thenComparingInt(ToIntFunction<? super T> keyExtractor) {
        return thenComparing(comparingInt(keyExtractor));
    }

    ...

    //------以下是所有的静态方法，可以直接生成一个新的Comparator对象
      
    //生成一个指定类型（需要实现Comparable）的反向比较器
    public static <T extends Comparable<? super T>> Comparator<T> reverseOrder() {
        return Collections.reverseOrder();
    }

    //生成一个指定类型（需要实现Comparable）的正向比较器
    @SuppressWarnings("unchecked")
    public static <T extends Comparable<? super T>> Comparator<T> naturalOrder() {
        return (Comparator<T>) Comparators.NaturalOrderComparator.INSTANCE;
    }

    //生成一个指定类型的正向比较器，且针对于null值，会优先排到前面
    public static <T> Comparator<T> nullsFirst(Comparator<? super T> comparator) {
        return new Comparators.NullComparator<>(true, comparator);
    }

    //生成一个指定类型的正向比较器，且针对于null值，会直接排到后面
    public static <T> Comparator<T> nullsLast(Comparator<? super T> comparator) {
        return new Comparators.NullComparator<>(false, comparator);
    }

    //传入一个指定类型用于比较的Key的获取方式，然后再传入Key的比较操作实现
    public static <T, U> Comparator<T> comparing(
            Function<? super T, ? extends U> keyExtractor,
            Comparator<? super U> keyComparator)
    {
        Objects.requireNonNull(keyExtractor);
        Objects.requireNonNull(keyComparator);
        return (Comparator<T> & Serializable)
            (c1, c2) -> keyComparator.compare(keyExtractor.apply(c1),
                                              keyExtractor.apply(c2));
    }

    //传入一个指定类型用于比较的Key（需要实现Comparable）的获取方式，然后自动进行比较
    public static <T, U extends Comparable<? super U>> Comparator<T> comparing(
            Function<? super T, ? extends U> keyExtractor)
    {
        Objects.requireNonNull(keyExtractor);
        return (Comparator<T> & Serializable)
            (c1, c2) -> keyExtractor.apply(c1).compareTo(keyExtractor.apply(c2));
    }

    //针对于内置类型的
    public static <T> Comparator<T> comparingInt(ToIntFunction<? super T> keyExtractor) {
        Objects.requireNonNull(keyExtractor);
        return (Comparator<T> & Serializable)
            (c1, c2) -> Integer.compare(keyExtractor.applyAsInt(c1), keyExtractor.applyAsInt(c2));
    }

    ...
}
```

至此，有关两个比较相关的接口就介绍完成了。

### Collections工具类

我们在前面介绍了Arrays，它是一个用于操作数组的工具类，它给我们提供了大量的工具方法。

既然数组操作都这么方便了，集合操作能不能也安排点高级的玩法呢？那必须的，JDK为我们准备的Collocations类就是专用于集合的工具类，比如我们想快速求得List中的最大值和最小值：

```java
public static void main(String[] args) {
    List<Integer> list = new ArrayList<>();
    Collections.max(list);
    Collections.min(list);
}
```

同样的，我们可以对一个集合进行二分搜索（注意，集合的具体类型，必须是实现Comparable接口的类）：

```java
public static void main(String[] args) {
    List<Integer> list = Arrays.asList(2, 3, 8, 9, 10, 13);
    System.out.println(Collections.binarySearch(list, 8));
}
```

我们也可以对集合的元素进行快速填充，注意这个填充是对集合中已有的元素进行覆盖：

```java
public static void main(String[] args) {
    List<Integer> list = new ArrayList<>(Arrays.asList(1,2,3,4,5));
    Collections.fill(list, 6);
    System.out.println(list);
}
```

如果集合中本身没有元素，那么`fill`操作不会生效。

有些时候我们可能需要生成一个空的集合类返回，那么我们可以使用`emptyXXX`来快速生成一个只读的空集合：

```java
public static void main(String[] args) {
    List<Integer> list = Collections.emptyList();
  	//Collections.singletonList() 会生成一个只有一个元素的List
    list.add(10);   //不支持，会直接抛出异常
}
```

我们也可以将一个可修改的集合变成只读的集合：

```java
public static void main(String[] args) {
    List<Integer> list = new ArrayList<>(Arrays.asList(1,2,3,4,5));
    List<Integer> newList = Collections.unmodifiableList(list);
    newList.add(10);   //不支持，会直接抛出异常
}
```

我们也可以寻找子集合的位置：

```java
public static void main(String[] args) {
    List<Integer> list = new ArrayList<>(Arrays.asList(1,2,3,4,5));
    System.out.println(Collections.indexOfSubList(list, Arrays.asList(4, 5)));
}
```

得益于泛型的类型擦除机制，实际上最后只要是Object的实现类都可以保存到集合类中，那么就会出现这种情况：

```java
public static void main(String[] args) {
  	//使用原始类型接收一个Integer类型的ArrayList
    List list = new ArrayList<>(Arrays.asList(1,2,3,4,5));
    list.add("aaa");   //我们惊奇地发现，这玩意居然能存字符串进去
    System.out.println(list);
}
```

![image-20221004001007854](https://s2.loli.net/2022/10/04/FP5z3X8SEMkGYtT.png)

没错，由于泛型机制上的一些漏洞，实际上对应类型的集合类有可能会存放其他类型的值，泛型的类型检查只存在于编译阶段，只要我们绕过这个阶段，在实际运行时，并不会真的进行类型检查，要解决这种问题很简单，就是在运行时进行类型检查：

```java
public static void main(String[] args) {
    List list = new ArrayList<>(Arrays.asList(1,2,3,4,5));
    list = Collections.checkedList(list, Integer.class);   //这里的.class关键字我们会在后面反射中介绍，表示Integer这个类型
  	list.add("aaa");
    System.out.println(list);
}
```

`checkedXXX`可以将给定集合类进行包装，在运行时同样会进行类型检查，如果通过上面的漏洞插入一个本不应该是当前类型集合支持的类型，那么会直接抛出类型转换异常：

![image-20221004001409799](https://s2.loli.net/2022/10/04/5BHq1u9JU3bhdI6.png)

是不是感觉这个工具类好像还挺好用的？实际上在我们的开发中，这个工具类也经常被使用到。

### (Java 9) 集合工厂方法

有些时候我们可能需要创建一个携带默认数据的集合对象，比如一个水果清单：

```java
List<String> fruit = new ArrayList<>();
fruit.add("apple");
fruit.add("banana");
fruit.add("orange");
```

像这样一个一个添加实在是太累了，我们前面学习了`Arrays`工具类，它可以快速用于创建：

```java
List<String> fruit = Arrays.asList("apple", "banana", "orange");
```

不过这种操作实际上更像是官方为数组设计的工具。在Java9之后，官方为我们提供了更加方便的工厂方法进行快速创建，不仅仅是List、包括Map、Set都被添加了若干个名字为`of`的静态方法：

![image-20250707205131728](https://s2.loli.net/2025/07/07/YytpnaFwL5XACiM.png)

通过这种方式就能像这样编写了：

```java
List<String> fruits = List.of("apple", "banana", "orange");
Set<String> fruitSet = Set.of("apple", "banana", "orange");
```

通过这种方式得到的集合对象同样是只读的，不可对其内容进行修改。其中`Set`和`List`的使用方法与之前提到的`Arrays`类似，直接依次填写即可。注意这里的Set集合默认使用的是类似HashSet的实现，元素没有特定顺序。

对于Map来说，我们需要把key和value交替进行编写，比如：

```java
Map<Integer, String> fruitMap = Map.of(1, "apple", 2, "banana", 3, "orange");
```

其中前者为Key，后者为Value，使用方式与上面是差不多的。除了直接传入键值对，也可以传入Entry对象进行构建：

![image-20250707205406929](https://s2.loli.net/2025/07/07/ol4TyBChMN6R1ut.png)

### (Java 21) 有序集合功能规范

前面我们提到过，针对于有序集合类，Java 21新增了一系列`SequencedCollection`、`SequencedSet`和`SequencedMap`接口，这些接口扩展和规范了所有有序集合的功能，我们先来看比较简单的，我们以List列表为例：

```java
List<String> list = List.of("A", "B", "C");

System.out.println(list.get(0));   //Java21之前获取首个元素
System.out.println(list.getFirst());  //现在可以直接使用getFirst拿第一个了

System.out.println(list.get(list.size() - 1));  //Java21之前获取最后一个元素，还得先拿到总长度
System.out.println(list.getLast());   //现在直接能拿了，堪称史诗级更新
```

原本只有LinkedList才有的列表首尾操作功能，现在所有有序列表都可以使用了，算是真正统一了列表的使用方式。包括原本LinkedList作为栈和队列才有的首尾插入操作，现在对于所有有序列表都能使用了：

```java
List<String> list = new ArrayList<>();
list.addFirst("A");
list.addFirst("B");
```

其中比较重要的功能是我们可以直接生成对应列表的反转列表了，只需调用`reversed`方法：

```java
List<String> list = List.of("A", "B", "C");
System.out.println(list.reversed());
```

但是注意它并不是直接对原列表进行修改，而是直接返回了一个新的列表对象，其具体实现为`ReverseOrderListView`倒序列表视图，为什么叫这样一个名字呢？首先大家需要理解视图的概念，视图相当于是对原来的数据换了一种方式进行展示，他可以像普通的列表一样进行查询，也可以像普通的列表那样去修改，但是其并不存储任何数据，所有对于视图的修改最终会作用到原列表上。

这样就很好理解了，我们可以通过下面的例子来解释：

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C"));
List<String> rlist = list.reversed();
System.out.println(rlist);

list.addLast("D");   //向原本列表的尾部插入元素D
System.out.println(rlist);
```

![image-20250709003608414](https://s2.loli.net/2025/07/09/nLO1SorCPdhWefH.png)

可以看到，在修改原本列表之后，其反转之后的列表视图打印的结果也跟着发生了变化。同样的，我们也可以直接对视图进行修改：

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C"));
List<String> rlist = list.reversed();
System.out.println(list);

rlist.addLast("D");   //向反转列表的尾部插入元素D，相当于在原列表首部插入元素D
System.out.println(list);
```

![image-20250709003954386](https://s2.loli.net/2025/07/09/A16JM4doHsXKtEv.png)

可以看到，我们向视图中进行的插入操作，最终作用到原本的列表上了，并且作用的效果也是相反的。因此，如果你只是单纯希望对列表内部元素进行倒序排序，可以考虑使用我们下面会讲解的`Collections`工具类，其中包含大量处理集合的操作。

而`SequencedSet`，它是对于`Set`的扩展，同样添加了头插尾插的操作，但是只有一部分Set集合类实现了此接口。

其中`SequencedMap`相关方法与上述内容类似，这里就不重复讲解了。

### (Java 8) Stream流

Java 8 API添加了一个新的操作，我们称为流Stream，可以让你以一种声明的方式处理数据。

Stream 使用一种类似用 SQL 语句从数据库查询数据的直观方式来提供一种对 Java 集合运算和表达的高阶抽象。Stream API可以极大提高Java程序员的生产力，让程序员写出高效率、干净、简洁的代码。这种风格将要处理的元素集合看作一种流， 流在管道中传输， 并且可以在管道的节点上进行处理， 比如筛选， 排序，聚合等。元素流在管道中经过中间操作（intermediate operation）的处理，最后由最终操作(terminal operation)得到前面处理的结果。

![image-20221003232832897](https://s2.loli.net/2022/10/03/r4AtmVRZ51y7uxd.png)

它看起来就像一个工厂的流水线一样，每一步都会对数据进行处理，乍一看好像没区别啊。我们来看看普通的处理和使用Stream处理有什么不一样：

```java
public static void main(String[] args) {
  	//这是普通的处理
    List<String> list = new ArrayList<>();
    list.add("A");
    list.add("B");
    list.add("C");
  
  	//移除为B的元素
  	Iterator<String> iterator = list.iterator();
    while (iterator.hasNext()){
        if(iterator.next().equals("B")) iterator.remove();
    }
  
  	//这是通过Stream的处理
    list = list     //链式调用
            .stream()    //获取流
            .filter(e -> !e.equals("B"))   //只允许所有不是B的元素通过流水线
            .collect(Collectors.toList());   //将流水线中的元素重新收集起来，变回List
    System.out.println(list);
}
```

看完这个例子之后，好像感觉除了写法高级了点之外，也没什么区别啊？这有啥用啊。虽然看起来是一样的，但是实际上它们的执行过程存在一定的差别。

同样是一批数据，直接编写操作对数据处理相当于吧这一批数据处理完之后，又开始下一个步骤继续处理这一批数据，每一步需要等到上一步结束之后，拿到上一批数据才能开始下一步，每一个流程依次进行：

![image-20250707230607130](https://s2.loli.net/2025/07/07/QRBxXyA9OlC7sFf.png)

而Stream则是提前编排好流程，每一个元素依次独立走完流水线，无需按批次完成：

![image-20250707230814486](https://s2.loli.net/2025/07/07/7oh9XCp5cK6vJlL.png)

前者需要在不同工厂之间传递整批数据，而后者则是直接把所有工厂揉在一起，形成流水线。

我们再通过一个例子来感受一下这种过程，下面的例子包含三个操作：

```java
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6);
list.stream()
        .map(x -> x)  //map可以对实现接受一种类型的数据，转为同类型或另一类型的数据
        .map(x -> x)  //这里为了简单好理解，就不做任何处理原样返回
        .forEach(System.out::println);  //最后一步是打印
```

现在我们在每一步操作上都添加一个打印：

```java
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6);
list.stream()
        .map(x -> {
            System.out.println("我是第一步: " + x);
            return x;
        })  //map可以对实现接受一种类型的数据，转为同类型或另一类型的数据
        .map(x -> {
            System.out.println("我是第二步: " + x);
            return x;
        })
        .forEach(System.out::println);
```

得到的结果为：

![image-20250707232051022](https://s2.loli.net/2025/07/07/RxgFLaSyft8Il6B.png)

乍一看好像有点看不懂啊，这顺序怎么好像是乱的呢，实则不然，还记得我们上面说的每个元素依次走流水线吗？我们可以来看前三步，首先是1号元素进入到第一步，接着进入到第二步，最后在第三步打印时结束。此时1号元素结束，紧接着又是2号元素在进行同样的流水线流程，这下就很清晰了，确实是依次按照流水线执行。

实际上，在我们调用Stream的各种方法时，并不是像我们之前那样直接对数据进行处理，而是在进行编排，直到我们调用最后一步`forEach`或是`collect`这种收尾的方法时，整个流水线才开始运作（不相信可以试试看去掉最后的收尾操作，不会有任何操作执行）这也是Stream的一种惰性的体现。

**注意：** 后续我们学习多线程之后，Stream甚至还能利用多线程，同时开展N条流水线进行处理，大大提升数据处理效率。

我们再来看一个例子，这个例子包含四个流水线操作：

```java
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6);
list.stream()
        .map(x -> {
            System.out.println("我是第一步: " + x);
            return x;
        })
        .filter(x -> x % 2 == 0)  //过滤掉所有奇数
        .map(x -> {
            System.out.println("我是第三步: " + x);
            return x;
        })
        .forEach(System.out::println);
```

![image-20250707232933301](https://s2.loli.net/2025/07/07/1reHCPgmLAMs7cX.png)

其中，由于第二步中进行了过滤操作，很多元素在进入到第三步之前就被提前丢弃了。这在处理大量数据的时候非常有效，一次性处理全部数据会占用非常多的内存，而通过流水线一步一步进行，依次进行数据处理会省去很多内存开销。

同时，利用流中提供的预设操作，我们无需自己编写繁琐的操作流程代码，只需要调用几个方法进行编排即可：

```java
public static void main(String[] args) {
    List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6);

    list = list
            .stream()
      			.distinct()   //去重（使用equals判断）
            .sorted((a, b) -> b - a)    //进行倒序排列
            .map(e -> e+1)    //每个元素都要执行+1操作
            .limit(2)    //只放行前两个元素
            .collect(Collectors.toList());

    System.out.println(list);
}
```

当遇到大量的复杂操作时，我们可以使用Stream来快速编写，这样不仅代码量大幅度减少，而且逻辑也更加清晰明了。

接下来，我们用一堆随机数来进行更多流操作的演示：

```java
public static void main(String[] args) {
    Random random = new Random();  //没想到吧，Random支持直接生成随机数的流
    random
            .ints(-100, 100)   //生成-100~100之间的，随机int型数字（本质上是一个IntStream）
            .limit(10)   //只获取前10个数字（这是一个无限制的流，如果不加以限制，将会无限进行下去！）
            .filter(i -> i < 0)   //只保留小于0的数字
            .sorted()    //默认从小到大排序
            .forEach(System.out::println);   //依次打印
}
```

值得注意的是，由于Stream是惰性的，每次只会让Random生成一个新的数据，直到达到`limit()`阈值，Random则不会再继续生产新的数据了。

利用StreamAPI提供的快捷操作，我们可以创建一个`IntSummaryStatistics`统计实例对象来帮助我们快速进行统计：

```java
public static void main(String[] args) {
    Random random = new Random();  //Random是一个随机数工具类
    IntSummaryStatistics statistics = random
            .ints(0, 100)
            .limit(100)
            .summaryStatistics();    //获取语法统计实例
    System.out.println(statistics.getMax());  //快速获取最大值
    System.out.println(statistics.getCount());  //获取数量
    System.out.println(statistics.getAverage());   //获取平均值
}
```

普通的List只需要一个方法就可以直接转换到方便好用的IntStream了：

```java
public static void main(String[] args) {
		List<String> list = Arrays.asList("AAA", "AAA", "AAAA", "AA");
    list.stream()
            .mapToInt(i -> i.length())    //将每一个元素映射为Integer类型
            .summaryStatistics();
}
```

我们还可以通过`flatMap`来对整个流进行进一步细分：

```java
public static void main(String[] args) {
    List<String> list = new ArrayList<>();
    list.add("A,B");
    list.add("C,D");
    list.add("E,F");   //我们想让每一个元素通过,进行分割，变成独立的6个元素
    list = list
            .stream()    //生成流
            .flatMap(e -> Arrays.stream(e.split(",")))    //分割字符串并生成新的流
            .collect(Collectors.toList());   //汇成新的List
    System.out.println(list);   //得到结果
}
```

我们也可以只通过Stream来完成所有数字的和，使用`reduce`方法：

```java
public static void main(String[] args) {
    List<Integer> list = new ArrayList<>();
    list.add(1);
    list.add(2);
    list.add(3);
    int sum = list
            .stream()
            .reduce((a, b) -> a + b)   //计算规则为：a是上一次计算的值，b是当前要计算的参数，这里是求和
            .get();    //我们发现得到的是一个Optional类实例，通过get方法返回得到的值
    System.out.println(sum);
}
```

可能，作为新手来说，一次性无法接受这么多内容，但是在各位以后的开发中，就会慢慢使用到这些东西了。

### (Java 9/11/16) Stream增强方法

由于Java 8推出的Stream获得了不错的反响，于是在Java 9中为其添加了更多方便实用的方法。集合类可以很轻松地转换为Stream：

```java
String[] strings = new String[]{"AAA", "AAA", "AA"};
List<String> list = Arrays.asList(strings);  //还得先转List
list.stream();
```

同样的，Java 8为我们提供了`of`方法用于快速将数组或单个元素包装为Steam对象：

```java
String[] strings = new String[]{"AAA", "AAA", "AA"};
Stream.of(strings).forEach(System.out::println);
```

在Java 9之后，对于可能为`null`的元素，我们也可以使用`ofNullable`方法来包装，如果发现传入的为`null`则返回一个空的Stream。

当然也新增了一些比较方便的操作，我们先来看`takeWhile`操作：

```java
List<String> list = Arrays.asList("AAA", "AAA", "AAAA", "AA");
list.stream()
        .takeWhile(s -> s.length() <= 3)   //不断获取直到不满足条件为止，与filter类似，但是会提前终止
        .forEach(System.out::println);
```

与之相反的还有`dropWhile`会尝试不断丢弃，直到第一个不满足条件的出现，才开始依次把元素往下一个流水线输送。

针对于String类型，我们也可以使用流的形式处理其中的每一个字符：

```java
str.chars()  //使用chars得到一个包含每个字符代码的IntStream
        .filter(c -> c == 'l')
        .forEach(System.out::println);
```

> 与其相似的还有一个codePoints方法，它可以按照按完整Unicode代码点处理，确保字符完整性和正确性，用于字符串中存在一些特殊字符的情况。

还有一个就是Java 11新增的`lines()`方法，它会自动对字符串按行分割，返回每一行的子串流。

在Java 16中，Stream中的操作被进一步强化，其中最有用的是常用的`List`收集被直接整合为了一个方法：

```java
String[] strings = new String[]{"AAA", "AAA", "AA"};
Stream.of(strings)
        .filter(s -> s.length() > 2)
        .toList();   //Java 16之后可以直接使用toList而不是collect
```

除此之外，还有`mapMulti`以及其衍生方法，不过这个方法效果与`flatMap`相似，用法传参相对比较复杂，在一些特殊情况下使用，这里就不进行介绍了，感兴趣可以自行了解。

### (Java 24) 流聚集器

传统Stream操作是无状态的，每个元素处理的流程单独进行，而Gatherers可以在处理过程中的同时维护状态，让前面元素的处理结果影响后续元素的处理方式，这里我们来尝试一下官方预设的一些操作。

现在给你一个需求，我们现在需要将包含`[A, B, C, D]`的流进行两两分组，得到`[[A, B],[C, D]]`然后再处理成每个分组内两个字符串合并的字符串，最后得到`[AB, CD]`，现在你该怎么做？

```java
List<String> list = List.of("A", "B", "C", "D");
list.stream()
        //需要使用Collectors的groupingBy工具进行分组，得到Map
        .collect(Collectors.groupingBy(s -> list.indexOf(s) / 2))
  			//把Map的值全部取出来，在转为stream
        .values()
        .stream()
        .map(group -> group.get(0) + group.get(1))
        .forEach(System.out::println);
```

可以看到，在之前要对数据进行动态处理，实在是太累了，又有了Gatherers之后，我们可以像这样写：

```java
List<String> list = List.of("A", "B", "C", "D");
list.stream()
        .gather(Gatherers.windowFixed(2))  //使用Gatherers的windowFixed创建固定窗口，元素将按照窗口大小自动进行分组
        .map(item -> item.getFirst() + item.getLast())
        .forEach(System.out::println);
```

其中`windowFixed`就实现了对元素的有状态操作，它不断接受到来的元素，并将其暂存，达到窗口大小之后作为一个新的元素打包返回。

我们也可以自行编写一个聚集器来感受这种带状态的流操作，同样是实现上面的操作：

```java
List<String> tempList = new ArrayList<>();   //临时使用一个外部的数组保存临时结果
list.stream()
        .gather(Gatherer.of((_, item, downstream) -> {  //要求传入Integrator函数式接口，它用于依次处理流中每个元素
            tempList.add(item);   //其中item就是当前元素，downstream代表下游，用于继续向下游发送元素
            if(tempList.size() < 2) {  //如果临时列表元素不足2个，则继续缓存
                return true;   //返回true表示就行处理下一个，false就终止
            } else {
                downstream.push(List.copyOf(tempList));  //当元素存满2个时，向下游发送打包好的分组结果
                tempList.clear();   //清理，给后面来的接着用
                return downstream.isRejecting();  //这里需要判断下游是否还愿意继续接受新元素，因为有可能下游存在limit之类的操作，不需要再继续发送元素了，所以要返回判断结果
            }
        }))
        .forEach(System.out::println);
```

当然，这里只是简单演示Gatherer的功能，`Gatherer.of`还包含更多参数选项。

此外，官方还为我们提供了一些其他的预设的工具，比如`fold`，它是一个多对一的有状态收集器，有点类似于`reduce`的感觉，比如我们想实现列表去重功能：

```java
List<String> list = List.of("A", "B", "C", "D", "A", "C");
list.stream()    //让HashSet作为初始状态存储工具
        .gather(Gatherers.fold(HashSet::new, (set, item) -> {
            set.add(item);   //向Set中添加元素
            return set;   //操作完成后需要返回set用于下一轮
        }))
        .flatMap(Collection::stream)  //对得到的Set集合展平
        .forEach(System.out::println);
```

接着是`scan`，它会依次扫描流中每个元素，并会在下一轮中给出上一轮被扫描过的元素，这很适合那些需要用到上一轮计算结果的操作。比如我想让列表`[A, B, C, D]`变成`[A, AB, ABC, ABCD]`这种金字塔状，就可以使用这种操作完成：

```java
List<String> list = List.of("A", "B", "C", "D");
list.stream()
        .gather(Gatherers.scan(() -> "0", (current, item) -> current + item))
        .forEach(System.out::println);
```

![image-20250709221429476](https://s2.loli.net/2025/07/09/An9adO416pRmLEc.png)

最后还有一个`mapConcurrent`方法，它可以并发实现对元素的映射，相当于是多线程版本的`map`操作，有关多线程的内容，我们会放在后面的章节讲解。

***

## Java I/O

**注意：** 这块会涉及到**操作系统**和**计算机组成原理**相关内容。

I/O简而言之，就是输入输出，那么为什么会有I/O呢？其实I/O无时无刻都在我们的身边，比如读取硬盘上的文件，网络文件传输，鼠标键盘输入，也可以是接受单片机发回的数据，而能够支持这些操作的设备就是I/O设备。

我们可以大致看一下整个计算机的总线结构：

![image-20221004002405375](https://s2.loli.net/2022/10/04/Q8JGeMprkgHsnPY.png)

常见的I/O设备一般是鼠标、键盘这类通过USB进行传输的外设或者是通过Sata接口或是M.2连接的硬盘。一般情况下，这些设备是由CPU发出指令通过南桥芯片间接进行控制，而不是由CPU直接操作。

而我们在程序中，想要读取这些外部连接的I/O设备中的内容，就需要将数据传输到内存中。而需要实现这样的操作，单单凭借一个小的程序是无法做到的，而操作系统（如：Windows/Linux/MacOS）就是专门用于控制和管理计算机硬件和软件资源的软件，我们需要读取一个IO设备的内容时，就可以向操作系统发出请求，由操作系统帮助我们来和底层的硬件交互以完成我们的读取/写入请求。

从读取硬盘文件的角度来说，不同的操作系统有着不同的文件系统（也就是文件在硬盘中的存储排列方式，如Windows就是NTFS、MacOS就是APFS），硬盘只能存储一个个0和1这样的二进制数据，至于0和1如何排列，各自又代表什么意思，就是由操作系统的文件系统来决定的。从网络通信角度来说，网络信号通过网卡等设备翻译为二进制信号，再交给系统进行读取，最后再由操作系统来给到程序。

![image-20221004002733950](https://s2.loli.net/2022/10/04/13h7yTekm2FfnRw.png)

（传统的SATA硬盘就是通过SATA线与电脑主板相连，这样才可以读取到数据）

JDK提供了一套用于IO操作的框架，为了方便我们开发者使用，就定义了一个像水流一样，根据流的传输方向和读取单位，分为字节流InputStream和OutputStream以及字符流Reader和Writer的IO框架，当然，这里的Stream并不是前面集合框架认识的Stream，这里的流指的是数据流，通过流，我们就可以一直从流中读取数据，直到读取到尽头，或是不断向其中写入数据，直到我们写入完成，而这类IO就是我们所说的BIO，

字节流一次读取一个字节，也就是一个`byte`的大小，而字符流顾名思义，就是一次读取一个字符，也就是一个`char`的大小（在读取纯文本文件的时候更加适合），有关这两种流，会在后面详细介绍，这个章节我们需要学习16个关键的流。

### 文件字节流

要学习和使用IO，首先就要从最易于理解的读取文件开始说起。

首先介绍一下FileInputStream，我们可以通过它来获取文件的输入流：

```java
public static void main(String[] args) {
    try {   //注意，IO相关操作会有很多影响因素，有可能出现异常，所以需要明确进行处理
        FileInputStream inputStream = new FileInputStream("路径");
        //路径支持相对路径和绝对路径
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    }
}
```

相对路径是在当前运行目录（就是你在哪个目录运行java命令启动Java程序的）的路径下寻找文件，而绝对路径，是从根目录开始寻找。路径分割符支持使用`/`或是`\\`，但是不能写为`\`因为它是转义字符！比如在Windows下：

```
C://User/lbw/nb    这个就是一个绝对路径，因为是从盘符开始的
test/test          这个就是一个相对路径，因为并不是从盘符开始的，而是一个直接的路径
```

在Linux和MacOS下：

```
/root/tmp       这个就是一个绝对路径，绝对路径以/开头
test/test       这个就是一个相对路径，不是以/开头的
```

当然，这个其实还是很好理解的，我们在使用时注意一下就行了。

在使用完成一个流之后，必须关闭这个流来完成对资源的释放，否则资源会被一直占用：

```java
public static void main(String[] args) {
    FileInputStream inputStream = null;    //定义可以先放在try外部
    try {
        inputStream = new FileInputStream("路径");
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } finally {
        try {    //建议在finally中进行，因为关闭流是任何情况都必须要执行的！
            if(inputStream != null) inputStream.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

虽然这样的写法才是最保险的，但是显得过于繁琐了，尤其是finally中再次嵌套了一个try-catch块，因此在JDK1.7新增了try-with-resource语法，用于简化这样的写法（本质上还是和这样的操作一致，只是换了个写法）

```java
public static void main(String[] args) {

    //注意，这种语法只支持实现了AutoCloseable接口的类！
    try(FileInputStream inputStream = new FileInputStream("路径")) {   //直接在try()中定义要在完成之后释放的资源

    } catch (IOException e) {   //这里变成IOException是因为调用close()可能会出现，而FileNotFoundException是继承自IOException的
        e.printStackTrace();
    }
    //无需再编写finally语句块，因为在最后自动帮我们调用了close()
}
```

之后为了方便，我们都使用此语法进行教学。

现在我们拿到了文件的输入流，那么怎么才能读取文件里面的内容呢？我们可以使用`read`方法：

```java
public static void main(String[] args) {
    //test.txt：a
    try(FileInputStream inputStream = new FileInputStream("test.txt")) {
        //使用read()方法进行字符读取
        System.out.println((char) inputStream.read());  //读取一个字节的数据（英文字母只占1字节，中文占2字节）
        System.out.println(inputStream.read());   //唯一一个字节的内容已经读完了，再次读取返回-1表示没有内容了
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

使用read可以直接读取一个字节的数据，注意，流的内容是有限的，读取一个少一个。我们如果想一次性全部读取的话，可以直接使用一个while循环来完成：

```java
public static void main(String[] args) {
    //test.txt：abcd
    try(FileInputStream inputStream = new FileInputStream("test.txt")) {
        int tmp;
        while ((tmp = inputStream.read()) != -1){   //通过while循环来一次性读完内容
            System.out.println((char)tmp);
        }
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

使用`available`方法能查看当前可读的剩余字节数量（注意：并不一定真实的数据量就是这么多，尤其是在网络I/O操作时，这个方法只能进行一个预估也可以说是暂时能一次性可以读取的数量，当然在磁盘IO下，一般情况都是真实的数据量）

```java
try(FileInputStream inputStream = new FileInputStream("test.txt")) {
    System.out.println(inputStream.available());  //查看剩余数量
}catch (IOException e){
    e.printStackTrace();
}
```

当然，一个一个读取效率太低了，那能否一次性全部读取呢？我们可以预置一个合适容量的byte[]数组来存放：

```java
public static void main(String[] args) {
    //test.txt：abcd
    try(FileInputStream inputStream = new FileInputStream("test.txt")) {
        byte[] bytes = new byte[inputStream.available()];   //我们可以提前准备好合适容量的byte数组来存放
        System.out.println(inputStream.read(bytes));   //一次性读取全部内容（返回值是读取的字节数）
        System.out.println(new String(bytes));   //通过String(byte[])构造方法得到字符串
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

也可以控制要读取数量：

```java
System.out.println(inputStream.read(bytes, 1, 2));   //第二个参数是从给定数组的哪个位置开始放入内容，第三个参数是读取流中的字节数
```

**注意**：一次性读取同单个读取一样，当没有任何数据可读时，依然会返回-1

通过`skip()`方法可以跳过指定数量的字节，返回值为实际跳过的字节：

```java
public static void main(String[] args) {
    //test.txt：abcd
    try(FileInputStream inputStream = new FileInputStream("test.txt")) {
        System.out.println(inputStream.skip(1));
        System.out.println((char) inputStream.read());   //跳过了一个字节
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

注意：FileInputStream是不支持`reset()`的，虽然有这个方法，但是这里先不提及。

既然有输入流，那么文件输出流也是必不可少的：

```java
public static void main(String[] args) {
    //输出流也需要在最后调用close()方法，并且同样支持try-with-resource
    try(FileOutputStream outputStream = new FileOutputStream("output.txt")) {
        //注意：若此文件不存在，会直接创建这个文件！
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

输出流没有`read()`操作而是`write()`操作，使用方法同输入流一样，只不过现在的方向变为我们向文件里写入内容：

```java
public static void main(String[] args) {
    try(FileOutputStream outputStream = new FileOutputStream("output.txt")) {
        outputStream.write('c');   //同read一样，可以直接写入内容
      	outputStream.write("lbwnb".getBytes());   //也可以直接写入byte[]
      	outputStream.write("lbwnb".getBytes(), 0, 1);  //同上输入流
      	outputStream.flush();  //建议在最后执行一次刷新操作（强制写入）来保证数据正确写入到硬盘文件中
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

那么如果是我只想在文件尾部进行追加写入数据呢？我们可以调用另一个构造方法来实现：

```java
public static void main(String[] args) {
    try(FileOutputStream outputStream = new FileOutputStream("output.txt", true)) {  //true表示开启追加模式
        outputStream.write("lb".getBytes());   //现在只会进行追加写入，而不是直接替换原文件内容
        outputStream.flush();
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

利用输入流和输出流，就可以轻松实现文件的拷贝了：

```java
public static void main(String[] args) {
    try(FileOutputStream outputStream = new FileOutputStream("output.txt");
        FileInputStream inputStream = new FileInputStream("test.txt")) {   //可以写入多个
        byte[] bytes = new byte[10];    //使用长度为10的byte[]做传输媒介
        int tmp;   //存储本地读取字节数
        while ((tmp = inputStream.read(bytes)) != -1){   //直到读取完成为止
            outputStream.write(bytes, 0, tmp);    //写入对应长度的数据到输出流
        }
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

### 文件字符流

字符流不同于字节，字符流是以一个具体的字符进行读取，因此它只适合读纯文本的文件，如果是其他类型的文件不适用：

```java
public static void main(String[] args) {
    try(FileReader reader = new FileReader("test.txt")){
      	reader.skip(1);   //现在跳过的是一个字符
        System.out.println((char) reader.read());   //现在是按字符进行读取，而不是字节，因此可以直接读取到中文字符
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

同理，字符流只支持`char[]`类型作为存储：

```java
public static void main(String[] args) {
    try(FileReader reader = new FileReader("test.txt")){
        char[] str = new char[10];
        reader.read(str);
        System.out.println(str);   //直接读取到char[]中
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

既然有了Reader肯定也有Writer：

```java
public static void main(String[] args) {
    try(FileWriter writer = new FileWriter("output.txt")){
      	writer.getEncoding();   //支持获取编码（不同的文本文件可能会有不同的编码类型）
       writer.write('牛');
       writer.append('牛');   //其实功能和write一样
      	writer.flush();   //刷新
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

我们发现不仅有`write()`方法，还有一个`append()`方法，但是实际上他们效果是一样的，看源码：

```java
public Writer append(char c) throws IOException {
    write(c);
    return this;
}
```

append支持像StringBuilder那样的链式调用，返回的是Writer对象本身。

**练习**：尝试一下用Reader和Writer来拷贝纯文本文件。

这里需要额外介绍一下File类，它是专门用于表示一个文件或文件夹，只不过它只是代表这个文件，但并不是这个文件本身。通过File对象，可以更好地管理和操作硬盘上的文件。

```java
public static void main(String[] args) {
    File file = new File("test.txt");   //直接创建文件对象，可以是相对路径，也可以是绝对路径
    System.out.println(file.exists());   //此文件是否存在
    System.out.println(file.length());   //获取文件的大小
    System.out.println(file.isDirectory());   //是否为一个文件夹
    System.out.println(file.canRead());   //是否可读
    System.out.println(file.canWrite());   //是否可写
    System.out.println(file.canExecute());   //是否可执行
  
    file.createNewFile();   //创建新文件
    file.delete();   //删除文件
    file.mkdir();    //创建目录，但父级目录不存在会直接报错
    file.mkdirs();   //创建目录，如果父级目录不存在会一起创建
}
```

通过File对象，我们就能快速得到文件的所有信息，如果是文件夹，还可以获取文件夹内部的文件列表等内容：

```java
File file = new File("/");
System.out.println(Arrays.toString(file.list()));   //快速获取文件夹下的文件名称列表
for (File f : file.listFiles()){   //所有子文件的File对象
    System.out.println(f.getAbsolutePath());   //获取文件的绝对路径
}
```

如果我们希望读取某个文件的内容，可以直接将File作为参数传入字节流或是字符流：

```java
File file = new File("test.txt");
try (FileInputStream inputStream = new FileInputStream(file)){   //直接做参数
    System.out.println(inputStream.available());
}catch (IOException e){
    e.printStackTrace();
}
```

**练习**：尝试拷贝文件夹下的所有文件到另一个文件夹

### (Java 7/8/11) 文件工具类

`Files`类是`java.nio.file`包中的一个实用类，在Java 7中推出，提供了许多静态方法，用于文件和目录的操作。它大大简化了文件处理的工作，例如创建、删除、读取、写入和属性管理等。比如之前的创建目录操作：

```java
public static void main(String[] args) throws IOException {
    Files.createDirectory(Path.of("./test"));  //使用createDirectory直接创建一个指定目录
    Files.createDirectories(Path.of("./test"));
}
```

其中提供了非常方便的方法直接创建目录，只不过此工具类的路径需要配合`Path`对象来指定，它代表一个文件路径，可以直接通过`of`方法来创建。

```java
Path path = Path.of("src/com/test");
System.out.println(path.toAbsolutePath());   //获取绝对路径
System.out.println(path.toAbsolutePath().getParent());  //获取父路径
```

![image-20250708204944405](https://s2.loli.net/2025/07/08/1YIRTzM3nWwL8xK.png)

利用此类型代表路径，可以为我们带来更多的便利，如快速获取绝对路径、父级目录等。现在回到`Files`类，对于文件的删除也是很方便的：

```java
Path path = Path.of("test");
Files.delete(path);   //删除，没有文件报错
Files.deleteIfExists(path);  //删除，没有文件返回false否则返回true
```

还有很多对于文件的快捷操作：

```java
Files.readString(path)   //Java 11新增，读取所有内容并以字符串返回
List<String> lines = Files.readAllLines(path);   //一键读取所有文件内容并按行分割返回
Stream<String> linesStream = Files.lines(path);   //同上，以Java 8之后的Stream形式返回

Files.write(path, "HelloWorld".getBytes());   //一键写入内容
Files.writeString(path, "HelloWorld");  //Java 11新增，更快捷的用字符串一键写入

Files.copy(path, Path.of("another"));   //拷贝文件到另一个路径
Files.move(path, Path.of("another"));   //移动文件到另一个路径

Files.exists(path);   //判断文件是否存在
Files.notExists(path);  //判断文件是否不存在

Files.isExecutable(path);  //文件是否可执行
Files.isDirectory(path);  //文件是否是文件夹
Files.isWritable(path);   //文件是否可写
Files.isReadable(path);   //文件是否可读
Files.isHidden(path);   //文件是否为隐藏文件

Files.newInputStream(path);  //创建新的文件输入流
Files.newOutputStream(path);   //创建新的文件输出流
```

此外，对于文件的查找，该工具类也给了非常好用API，其中`find`方法可以帮助我们快速查找文件：

```java
//其中第一个参数为起始查找点，第二参数为最大查找深度，最后就是一个断言函数式，就是判断每一个文件是否符合我们要查找的要求
Files.find(Path.of("."), 4, (path, attributes) -> {
    if (path.getFileName().toString().equals("Main.java")) {
        return true;
    } else {
        return false;
    }
}).forEach(path -> {   //find的结果会以Stream的形式返回
    System.out.println(path);
});
```

对于整个目录下文件的遍历，我们可以使用`walkFileTree`方法，此方法需要我们手动实现一个`FileVisitor`类，用于指定文件的查找操作，假设我们希望遍历项目目录下所有文件并打印所有非文件夹的普通文件：

```java
//第一个参数指定起始位置，第二个参数实现FileVisitor
Files.walkFileTree(Path.of("."), new SimpleFileVisitor<>() {
    @Override
    public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) throws IOException {
        System.out.println("找到目录: " + dir.toString());
        return FileVisitResult.CONTINUE;  //需要返回一个文件结果，CONTINUE就是继续，如果不想继续了也可以返回TERMINATE或是SKIP_SUBTREE跳过这个目录
    }

    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
        System.out.println("找到文件: " + file.toString());
        return FileVisitResult.CONTINUE;
    }

    @Override
    public FileVisitResult visitFileFailed(Path file, IOException exc) throws IOException {
        System.err.println("访问失败: " + file.toString());
        return FileVisitResult.CONTINUE;
    }
});
```

不过上面这种方式写起来还是有点复杂，在Java8之后，Files也为我们提供了很方便的`walk`方法，它同样是返回一个Stream给我们，其中包含遍历的每一个文件的`Path`对象，这样就简单多了：

```java
Files.walk(Path.of("."))  //返回Stream<Path>
        .filter(Files::isRegularFile)   //filter过滤判断是否为普通文件
        .forEach(System.out::println);   //打印
```

有关Files的相关操作就介绍到这里，详细内容各位小伙伴可以继续尝试探索。

### 缓冲流

虽然普通的文件流读取文件数据非常便捷，但是每次都需要从外部I/O设备去获取数据，由于外部I/O设备的速度一般都达不到内存的读取速度，很有可能造成程序反应迟钝，因此性能还不够高，而缓冲流正如其名称一样，它能够提供一个缓冲，提前将部分内容存入内存（缓冲区）在下次读取时，如果缓冲区中存在此数据，则无需再去请求外部设备。同理，当向外部设备写入数据时，也是由缓冲区处理，而不是直接向外部设备写入。

![image-20221004125755217](https://s2.loli.net/2022/10/04/S8O61JP2lqKTzjd.png)

要创建一个缓冲字节流，只需要将原本的流作为构造参数传入BufferedInputStream即可：

```java
public static void main(String[] args) {
    try (BufferedInputStream bufferedInputStream = new BufferedInputStream(new FileInputStream("test.txt"))){   //传入FileInputStream
        System.out.println((char) bufferedInputStream.read());   //操作和原来的流是一样的
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

实际上进行I/O操作的并不是BufferedInputStream，而是我们传入的FileInputStream，而BufferedInputStream虽然有着同样的方法，但是进行了一些额外的处理然后再调用FileInputStream的同名方法，这样的写法称为`装饰者模式`，我们会在设计模式篇中详细介绍。我们可以来观察一下它的`close`方法源码：

```java
public void close() throws IOException {
    byte[] buffer;
    while ( (buffer = buf) != null) {
        if (bufUpdater.compareAndSet(this, buffer, null)) {  //CAS无锁算法，并发会用到，暂时不需要了解
            InputStream input = in;
            in = null;
            if (input != null)
                input.close();
            return;
        }
        // Else retry in case a new buf was CASed in fill()
    }
}
```

实际上这种模式是父类FilterInputStream提供的规范，后面我们还会讲到更多FilterInputStream的子类。

我们可以发现在BufferedInputStream中还存在一个专门用于缓存的数组：

```java
/**
 * The internal buffer array where the data is stored. When necessary,
 * it may be replaced by another array of
 * a different size.
 */
protected volatile byte buf[];
```

I/O操作一般不能重复读取内容（比如键盘发送的信号，主机接收了就没了），而缓冲流提供了缓冲机制，一部分内容可以被暂时保存，BufferedInputStream支持`reset()`和`mark()`操作，首先我们来看看`mark()`方法的介绍：

```java
/**
 * Marks the current position in this input stream. A subsequent
 * call to the <code>reset</code> method repositions this stream at
 * the last marked position so that subsequent reads re-read the same bytes.
 * <p>
 * The <code>readlimit</code> argument tells this input stream to
 * allow that many bytes to be read before the mark position gets
 * invalidated.
 * <p>
 * This method simply performs <code>in.mark(readlimit)</code>.
 *
 * @param   readlimit   the maximum limit of bytes that can be read before
 *                      the mark position becomes invalid.
 * @see     java.io.FilterInputStream#in
 * @see     java.io.FilterInputStream#reset()
 */
public synchronized void mark(int readlimit) {
    in.mark(readlimit);
}
```

当调用`mark()`之后，输入流会以某种方式保留之后读取的`readlimit`数量的内容，当读取的内容数量超过`readlimit`则之后的内容不会被保留，当调用`reset()`之后，会使得当前的读取位置回到`mark()`调用时的位置。

```java
public static void main(String[] args) {
    try (BufferedInputStream bufferedInputStream = new BufferedInputStream(new FileInputStream("test.txt"))){
        bufferedInputStream.mark(1);   //只保留之后的1个字符
        System.out.println((char) bufferedInputStream.read());
        System.out.println((char) bufferedInputStream.read());
        bufferedInputStream.reset();   //回到mark时的位置
        System.out.println((char) bufferedInputStream.read());
        System.out.println((char) bufferedInputStream.read());
    }catch (IOException e) {
        e.printStackTrace();
    }
}
```

我们发现虽然后面的部分没有保存，但是依然能够正常读取，其实`mark()`后保存的读取内容是取`readlimit`和BufferedInputStream类的缓冲区大小两者中的最大值，而并非完全由`readlimit`确定。因此我们限制一下缓冲区大小，再来观察一下结果：

```java
public static void main(String[] args) {
    try (BufferedInputStream bufferedInputStream = new BufferedInputStream(new FileInputStream("test.txt"), 1)){  //将缓冲区大小设置为1
        bufferedInputStream.mark(1);   //只保留之后的1个字符
        System.out.println((char) bufferedInputStream.read());
        System.out.println((char) bufferedInputStream.read());   //已经超过了readlimit，继续读取会导致mark失效
        bufferedInputStream.reset();   //mark已经失效，无法reset()
        System.out.println((char) bufferedInputStream.read());
        System.out.println((char) bufferedInputStream.read());
    }catch (IOException e) {
        e.printStackTrace();
    }
}
```

了解完了BufferedInputStream之后，我们再来看看BufferedOutputStream，其实和BufferedInputStream原理差不多，只是反向操作：

```java
public static void main(String[] args) {
    try (BufferedOutputStream outputStream = new BufferedOutputStream(new FileOutputStream("output.txt"))){
        outputStream.write("lbwnb".getBytes());
        outputStream.flush();
    }catch (IOException e) {
        e.printStackTrace();
    }
}
```

操作和FileOutputStream一致，这里就不多做介绍了。

既然有缓冲字节流，那么肯定也有缓冲字符流，缓冲字符流和缓冲字节流一样，也有一个专门的缓冲区，BufferedReader构造时需要传入一个Reader对象：

```java
public static void main(String[] args) {
    try (BufferedReader reader = new BufferedReader(new FileReader("test.txt"))){
        System.out.println((char) reader.read());
    }catch (IOException e) {
        e.printStackTrace();
    }
}
```

使用和reader也是一样的，内部也包含一个缓存数组：

```java
private char cb[];
```

相比Reader更方便的是，它支持按行读取：

```java
public static void main(String[] args) {
    try (BufferedReader reader = new BufferedReader(new FileReader("test.txt"))){
        System.out.println(reader.readLine());   //按行读取
    }catch (IOException e) {
        e.printStackTrace();
    }
}
```

读取后直接得到一个字符串，当然，它还能把每一行内容依次转换为集合类提到的Stream流：

```java
public static void main(String[] args) {
    try (BufferedReader reader = new BufferedReader(new FileReader("test.txt"))){
        reader
                .lines()
                .limit(2)
                .distinct()
                .sorted()
                .forEach(System.out::println);
    }catch (IOException e) {
        e.printStackTrace();
    }
}
```

它同样也支持`mark()`和`reset()`操作：

```java
public static void main(String[] args) {
    try (BufferedReader reader = new BufferedReader(new FileReader("test.txt"))){
        reader.mark(1);
        System.out.println((char) reader.read());
        reader.reset();
        System.out.println((char) reader.read());
    }catch (IOException e) {
        e.printStackTrace();
    }
}
```

BufferedReader处理纯文本文件时就更加方便了，BufferedWriter在处理时也同样方便：

```java
public static void main(String[] args) {
    try (BufferedWriter reader = new BufferedWriter(new FileWriter("output.txt"))){
        reader.newLine();   //使用newLine进行换行
        reader.write("汉堡做滴彳亍不彳亍");   //可以直接写入一个字符串
      	reader.flush();   //清空缓冲区
    }catch (IOException e) {
        e.printStackTrace();
    }
}
```

合理使用缓冲流，可以大大提高我们程序的运行效率，只不过现在初学阶段，很少会有机会接触到实际的应用场景。

### 转换流

有时会遇到这样一个很麻烦的问题：我这里读取的是一个字符串或是一个个字符，但是我只能往一个OutputStream里输出，但是OutputStream又只支持byte类型，如果要往里面写入内容，进行数据转换就会很麻烦，那么能否有更加简便的方式来做这样的事情呢？

```java
public static void main(String[] args) {
    try(OutputStreamWriter writer = new OutputStreamWriter(new FileOutputStream("test.txt"))){  //虽然给定的是FileOutputStream，但是现在支持以Writer的方式进行写入
        writer.write("lbwnb");   //以操作Writer的样子写入OutputStream
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

同样的，我们现在只拿到了一个InputStream，但是我们希望能够按字符的方式读取，我们就可以使用InputStreamReader来帮助我们实现：

```java
public static void main(String[] args) {
    try(InputStreamReader reader = new InputStreamReader(new FileInputStream("test.txt"))){  //虽然给定的是FileInputStream，但是现在支持以Reader的方式进行读取
        System.out.println((char) reader.read());
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

InputStreamReader和OutputStreamWriter本质也是Reader和Writer，因此可以直接放入BufferedReader来实现更加方便的操作。

### 打印流

打印流其实我们从一开始就在使用了，比如`System.out`就是一个PrintStream，PrintStream也继承自FilterOutputStream类因此依然是装饰我们传入的输出流，但是它存在自动刷新机制，例如当向PrintStream流中写入一个字节数组后自动调用`flush()`方法。PrintStream也永远不会抛出异常，而是使用内部检查机制`checkError()`方法进行错误检查。最方便的是，它能够格式化任意的类型，将它们以字符串的形式写入到输出流。

```java
public final static PrintStream out = null;
```

可以看到`System.out`也是PrintStream，不过默认是向控制台打印，我们也可以让它向文件中打印：

```java
public static void main(String[] args) {
    try(PrintStream stream = new PrintStream(new FileOutputStream("test.txt"))){
        stream.println("lbwnb");   //其实System.out就是一个PrintStream
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

我们平时使用的`println`方法就是PrintStream中的方法，它会直接打印基本数据类型或是调用对象的`toString()`方法得到一个字符串，并将字符串转换为字符，放入缓冲区再经过转换流输出到给定的输出流上。

![img](https://s2.loli.net/2022/10/04/w8RKJxLm6Ik5usn.png)

因此实际上内部还包含这两个内容：

```java
/**
 * Track both the text- and character-output streams, so that their buffers
 * can be flushed without flushing the entire stream.
 */
private BufferedWriter textOut;
private OutputStreamWriter charOut;
```

与此相同的还有一个PrintWriter，不过他们的功能基本一致，PrintWriter的构造方法可以接受一个Writer作为参数，这里就不再做过多阐述了。

而我们之前使用的Scanner，使用的是系统提供的输入流：

```java
public static void main(String[] args) {
    Scanner scanner = new Scanner(System.in);   //系统输入流，默认是接收控制台输入
}
```

我们也可以使用Scanner来扫描其他的输入流：

```java
public static void main(String[] args) throws FileNotFoundException {
    Scanner scanner = new Scanner(new FileInputStream("秘制小汉堡.txt"));  //将文件内容作为输入流进行扫描
}
```

相当于直接扫描文件中编写的内容，同样可以读取。

### 数据流

数据流DataInputStream也是FilterInputStream的子类，同样采用装饰者模式，最大的不同是它支持基本数据类型的直接读取：

```java
public static void main(String[] args) {
    try (DataInputStream dataInputStream = new DataInputStream(new FileInputStream("test.txt"))){
        System.out.println(dataInputStream.readBoolean());   //直接将数据读取为任意基本数据类型
    }catch (IOException e) {
        e.printStackTrace();
    }
}
```

用于写入基本数据类型：

```java
public static void main(String[] args) {
    try (DataOutputStream dataOutputStream = new DataOutputStream(new FileOutputStream("output.txt"))){
        dataOutputStream.writeBoolean(false);
    }catch (IOException e) {
        e.printStackTrace();
    }
}
```

注意，写入的是二进制数据，并不是写入的字符串，使用DataInputStream可以读取，一般他们是配合一起使用的。

### 对象流

既然基本数据类型能够读取和写入基本数据类型，那么能否将对象也支持呢？ObjectOutputStream不仅支持基本数据类型，通过对对象的序列化操作，以某种格式保存对象，来支持对象类型的IO，注意：它不是继承自FilterInputStream的。

```java
public static void main(String[] args) {
    try (ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream("output.txt"));
         ObjectInputStream inputStream = new ObjectInputStream(new FileInputStream("output.txt"))){
        People people = new People("lbw");
        outputStream.writeObject(people);
      	outputStream.flush();
        people = (People) inputStream.readObject();
        System.out.println(people.name);
    }catch (IOException | ClassNotFoundException e) {
        e.printStackTrace();
    }
}

static class People implements Serializable{   //必须实现Serializable接口才能被序列化
    String name;

    public People(String name){
        this.name = name;
    }
}
```

在我们后续的操作中，有可能会使得这个类的一些结构发生变化，而原来保存的数据只适用于之前版本的这个类，因此我们需要一种方法来区分类的不同版本：

```java
static class People implements Serializable{
    private static final long serialVersionUID = 123456;   //在序列化时，会被自动添加这个属性，它代表当前类的版本，我们也可以手动指定版本。

    String name;

    public People(String name){
        this.name = name;
    }
}
```

当发生版本不匹配时，会无法反序列化为对象：

```java
java.io.InvalidClassException: com.test.Main$People; local class incompatible: stream classdesc serialVersionUID = 123456, local class serialVersionUID = 1234567
	at java.io.ObjectStreamClass.initNonProxy(ObjectStreamClass.java:699)
	at java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:2003)
	at java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1850)
	at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2160)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1667)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:503)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:461)
	at com.test.Main.main(Main.java:27)
```

如果我们不希望某些属性参与到序列化中进行保存，我们可以添加`transient`关键字：

```java
public static void main(String[] args) {
    try (ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream("output.txt"));
         ObjectInputStream inputStream = new ObjectInputStream(new FileInputStream("output.txt"))){
        People people = new People("lbw");
        outputStream.writeObject(people);
        outputStream.flush();
        people = (People) inputStream.readObject();
        System.out.println(people.name);  //虽然能得到对象，但是name属性并没有保存，因此为null
    }catch (IOException | ClassNotFoundException e) {
        e.printStackTrace();
    }
}

static class People implements Serializable{
    private static final long serialVersionUID = 1234567;

    transient String name;

    public People(String name){
        this.name = name;
    }
}
```

其实我们可以看到，在一些JDK内部的源码中，也存在大量的transient关键字，使得某些属性不参与序列化，取消这些不必要保存的属性，可以节省数据空间占用以及减少序列化时间。

### 字节数组流

这个流是一个非常特殊的流，它内部维护了一个`byte[]`类型的对象，用于直接包含流的数据。

有些时候我们可能希望将某些数据直接转换为流的形式，比如字符串：

```java
public static void main(String[] args) throws IOException {
    InputStream stream = new ByteArrayInputStream("我是一个可爱的字符串".getBytes());
    BufferedReader buffered = new BufferedReader(new InputStreamReader(stream));
    System.out.println(buffered.readLine());
}
```

通过构造`ByteArrayInputStream`来快速将我们需要的数据构造为流，它接受一个`byte[]`数组，此数组将直接作为其数据，当对流的数据进行读取时，直接返回数据中的内容。

与之相反的有一个叫`ByteArrayOutputStream`的流，它内部同样维护一个`byte[]`数组，用于存放外部写入的数据，在写入数据完成后，我们可以调用`toByteArray`来获取其中的数据：

```java
public static void main(String[] args) throws IOException {
    ByteArrayOutputStream stream = new ByteArrayOutputStream();
    stream.write("Hello, world!".getBytes());
    System.out.println(new String(stream.toByteArray()));
}
```

这两种流在处理一些直接数据时非常方便。

### (Java 9/11/12) 输入流快捷操作

在Java9之后，InputStream中新增了很多实用的工具方法，这里我们来学习一下。

首先是内容读取，在之前我们如果需要读取一个文本文件的内容，需要写一个循环来不断`read`里面的内容，直到结束，这实在是太繁琐了，通过`available()`获取的剩余容量又不一定准确。所以，从Java 9 开始，官方内置一个新的方法来便于我们一次性读取文件内容，我们终于可以解放双手了：

```java
try (InputStream stream = new FileInputStream("./hello.txt")) {
    byte[] bytes = stream.readAllBytes();   //此方法可以阻塞地一次性读取流中所有剩余字节
    System.out.println(new String(bytes));
} catch (IOException e) {
    e.printStackTrace();
}
```

![image-20250707214750947](https://s2.loli.net/2025/07/07/9jLtir2Shd7DWPu.png)

注意如果流已经被读取完成，调用此方法只能直接返回空数组，因为没数据可读了。

那如果我需要读取指定数量的数据怎么办呢？同样的，Java 9 为我们提供了`readNBytes`方法，它可以更灵活地控制读取流中的多少字节或是哪个位置开始的字节：

```java
try (InputStream stream = new FileInputStream("./hello.txt")) {
    byte[] bytes = new byte[9];
    stream.readNBytes(bytes, 0, bytes.length);  //从流的起始位置开始读取9个字节
    System.out.println(new String(bytes));
} catch (IOException e) {
    e.printStackTrace();
}
```

![image-20250707215146453](https://s2.loli.net/2025/07/07/14cyC8xMJeB5qYL.png)

注意`readNBytes`返回值为实际读取的字节数，因为有可能出现流中数据还没有想要的这么长的情况。

从Java 11开始，我们还可以直接使用一个重载之后的更简单的`readNBytes`方法：

```java
try (InputStream stream = new FileInputStream("./hello.txt")) {
    byte[] bytes = stream.readNBytes(9);   //和readAllBytes一样，直接返回数组
    System.out.println(new String(bytes));
} catch (IOException e) {
    e.printStackTrace();
}
```

它可以直接指定要读取的数量，并返回对应的byte数组。

对于值之前的`skip()`方法，Java 12为我们提供了一个更强大的`skipNBytes`方法，此方法同样用于跳过指定字节，但是其行为是**精确地跳过**输入流中的 `n` 个，它不像`skip()`那样跳过多少算多少，而是要求必须跳过这么多字节，否则抛出`EOFException`异常，比如：

```java
try (InputStream stream = new ByteArrayInputStream("大烟杆嘴里塞，我只抽第五代".getBytes())) {
    stream.skipNBytes(200);
} catch (IOException e) {
    e.printStackTrace();
}
```

Java 11还为我们提供了一个直接生成已读取结束的流的方法：

```java
try (InputStream stream = InputStream.nullInputStream()) { //直接生成一个没有数据的等价于用完的流
    byte[] bytes = stream.readAllBytes();
    System.out.println(new String(bytes));   //读取出来的直接就是一个空数组，因为已经用过了
} catch (IOException e) {
    e.printStackTrace();
}
```

类似的还有OutputStream的`nullOutputStream`方法，生成一个可以无限倾倒数据的输出流，就像给黑洞吞了就没了。

最后还有一个非常实用的方法，它是Java 9提供的`transferTo`，它可以直接将当前输入流中的内容转换到一个输出流中，相当于用一个管道直接把输入输出连接起来，输入的内容直接往输出的地方跑。这就很方便了，比如我们要拷贝一个文件，可以直接使用：

```java
try (InputStream in = new FileInputStream("./hello.txt");
     OutputStream out = new FileOutputStream("./hello2.txt")) {
    in.transferTo(out);   //直接transfer到新文件里，并返回实际传输了多少字节的数据
} catch (IOException e) {
    e.printStackTrace();
}
```

这样，一个文件拷贝就轻松实现了。

***

## 实战：图书管理系统

要求实现一个图书管理系统（控制台），支持以下功能：保存书籍信息（要求持久化），查询、添加、删除、修改书籍信息。

————————————————
版权声明：本文为柏码知识库版权所有，禁止一切未经授权的转载、发布、出售等行为，违者将被追究法律责任。
原文链接：https://www.itbaima.cn/zh-CN/document/k6fmxd6qabgkwm9i