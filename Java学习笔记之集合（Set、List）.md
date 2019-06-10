# `Java学习笔记——Set、List集合`

标签： `Java 集合（Set、List）`

---

## （一）`Set`

> * `Set`集合是最简单的一种集合，集合中的对象无序、不能重复。
> * 用得较多的主要实现类有`HashSet`和`TreeSet`。
> * `HashSet`：按照哈希算法来存取集合中的对象，存取速度比较快。
> * `TreeSet`：实现`SortedSet`接口，具有排序功能。

### （1）`HashSet`
`Set`集合没有重复对象，当一个新的对象加入到`HashSet`集合中时，`HashSet`的`add`方法会先判断是否存在这个对象，如果不是则添加到集合中。判断过程：先遍历集合，判断集合中有没有一个对象的`HashCode`与待添加的对象的`HashCode`相等，如果没有，则将待添加对象添加到集合中，如果有，则进一步判断`equals`，如果该对象与待添加对象的`equals`比较结果为`false`，则将待添加对象添加到集合中。
        
来看一个例子🌰：
```java
    String s1 = new String("123");
    String s2 = new String("123");
    Set<String> set = new HashSet<>();
    set.add(s1);
    set.add(s2);
    set.forEach(System.out::println); // Java8新特性，增强for循环
```
结果为`"123"`，说明s2并没有加入到`set`中，这是为什么呢，看下面给出的`JDK`源码，我们知道，`String`类是重写了`Object`类的`hashCode`方法和`equals`方法的，首先`s1`和`s2`的值都为`"123"`，通过下面的源码可以得出结论，它们的`hashCode`相等，`equals`（重写之后变为比较它们的值）为`true`，即视为同一个对象。如果再加入一个到集合中：
```java
String s3 = "123";
```
结果还是一个`"123"`，道理一样。
```java
//JDK String类的hashCode源码
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;
        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}

//JDK String类的equals源码
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
```

再看一个例子🌰：  
这次我们自己新建一个`Student`类:
```java
class Student {
	private String name;
	private int age;

	public Student(String name, int age) {
		super();
		this.name = name;
		this.age = age;
	}

	public Student() {
		super();
	}

	@Override
	public String toString() {
		return "Student [name=" + name + ", age=" + age + "]";
	}
}
```

然后再执行下面代码：
```java
    Set<Student> set1 = new HashSet<>();
	Student student1 = new Student("hzj", 1);
	Student student2 = new Student("hzj", 1);
	set1.add(student1);
	set1.add(student2);
	new HashSet<>().add(student1);
	set1.forEach(System.out::println);
```

结果为`Student [name=hzj, age=1] Student [name=hzj, age=1]`，有两个“一样的”值，原因很显然，`new`了两个对象，那么它们的`hashCode`值肯定不同。下面给`Student`类重写`hashCode`方法：
```java
@Override
public int hashCode() {
    return name.hashCode() + age;
}
```

添加之后结果还是两个，因为它们的`equals`结果为`false`，下面再给`Student`类重写`equals`方法：
```java
@Override
public boolean equals(Object obj) {
    if (this == obj)
        return true;
    if (obj instanceof Student) {
        Student student = (Student) obj;
        if (student.name.equals(name) && student.age == age) {
            return true;
        }
    }
    return false;
}
```

这样，结果就为一个了。

### （2）`TreeSet`
`TreeSet`实现了`SortedSet`接口，能够对集合中的对象进行排序。当`TreeSet`向集合中加入一个对象时，会把它插入到有序的对象序列中。`TreeSet`支持两种排序方式：自然排序和客户化排序。默认情况下`TreeSet`采用的是自然排序方式。

> * 自然排序  

使要排序的类实现`Comparable`接口并实现`compareTo`方法。例如：
```java
class Student implements Comparable<Student>{
	private String name;
	private int age;

	public Student(String name, int age) {
		super();
		this.name = name;
		this.age = age;
	}

	public Student() {
		super();
	}

	@Override
	public String toString() {
		return "Student [name=" + name + ", age=" + age + "]";
	}

	@Override
	public int compareTo(Student o) { // 通过名字的升序排列
		return name.compareTo(o.name);
	}
}
```
```java
    // 测试代码
    Set<Student> set1 = new TreeSet<>();
    Student student1 = new Student("b", 1);
    Student student2 = new Student("a", 2);
    Student student3 = new Student("c", 0);
    set1.add(student1);
    set1.add(student2);
    set1.add(student3);
    new HashSet<>().add(student1);
    set1.forEach(System.out::println);

    /*结果：名字升序
    Student [name=a, age=2]
    Student [name=b, age=1]
    Student [name=c, age=0]
    */
```

> * 客户化排序

`java.util.Comparator`接口提供了具体的排序方法， 它有一个`compare(Object x, Object y)`方法，用于比较两个对象的大小。  
此方式下的`Student`类不用实现`Comparable`接口，而是新建一个比较类，让这个类实现`Comparator`接口，详细代码：
```java
package com.birup.hdjd.chap06;

import java.util.Comparator;
import java.util.HashSet;
import java.util.Set;
import java.util.TreeSet;

public class CollectionTest {

	public static void main(String[] args) {
		Set<Student> set1 = new TreeSet<>(new StudentComparator());//注意这里与上面自然排序的区别，使用的构造器不一样
		Student student1 = new Student("b", 1);
		Student student2 = new Student("a", 2);
		Student student3 = new Student("c", 0);
		set1.add(student1);
		set1.add(student2);
		set1.add(student3);
		new HashSet<>().add(student1);
		set1.forEach(System.out::println);
	}
    /*结果：
    Student [name=c, age=0]
    Student [name=b, age=1]
    Student [name=a, age=2]
    */

}
class StudentComparator implements Comparator<Student>{
	@Override
	public int compare(Student o1, Student o2) {
		return o1.getAge() - o2.getAge(); // 年龄升序
	}
}
class Student{
	private String name;
	private int age;

	public Student(String name, int age) {
		super();
		this.name = name;
		this.age = age;
	}

	public Student() {
		super();
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}

	@Override
	public String toString() {
		return "Student [name=" + name + ", age=" + age + "]";
	}
}

```

## （二）`List` 

> * List的遍历方式有两种：
    a. `list.get(i);    //通过索引检索对象；`
    b. `Iterator it = list.iterator(); //通过迭代器`
       `it.next();`

> * 添加对象：`add(object)`
> * 获取对象：`get(index)`
> * 删除对象：`remove(index|object)`
> * 判断是否存在对象：`contains(Object)`


可以用`Arrays.asList()`将一个数组转化为一个`List`对象：
```java
	Integer[] a = new Integer[] {1, 3, 2, 5};
	List<Integer> asList = Arrays.asList(a);
	asList.forEach(System.out::println);
	/*结果：
		1
		3
		2
		5
	*/
```

也可以用`Collection.toArray()`将一个集合转换成一个数组对象：
```java
	List<Integer> asList = Arrays.asList(1, 2, 3, 4);
	Object[] array = asList.toArray();
	for (Object object : array)
		System.out.println(object.toString());
	/*结果：
		1
		2
		3
		4
	*/
```

### （1）`ArrayList`
> * `ArrayList` 代表长度可变的数组。
> * `ArrayList` 允许对元素进行快速的随机访问，但是向ArrayList中插入与删除元素的速度较慢。

使用方式：
```java
	List<Integer> list = new ArrayList<>();
	list.add(11);  // 添加
	list.add(22);
	list.remove(0);  // 删除remove(index|object)
	list.get(0);  // 取值get(index)
```

### （2）`LinkedList`
> * `LinkedList`插入是比较快的，因为不用移动元素。

使用方式：
```java
	List<Integer> list = new LinkedList<>();
	list.add(11);  // 添加
	list.add(22);
	list.remove(0);  // 删除remove(index|object)
	list.get(0);  // 取值get(index)
```

### （3）`ArrayList与LinkedList比较`

> * `LinkedList`和`ArrayList`的差别主要来自于`Array`和`LinkedList`数据结构的不同。
> * `Array`是基于索引`(index)`的数据结构，它使用索引在数组中搜索和读取数据是很快的。但是要删除数据却是开销很大的，因为这需要重排数组中的所有数据。
> * 相对于`ArrayList`，`LinkedList`插入是更快的。因为`LinkedList`不像`ArrayList`一样，不需要改变数组的大小，也不需要在数组装满的时候要将所有的数据重新装入一个新的数组，这是`ArrayList`最坏的一种情况。
> * `LinkedList`需要更多的内存。

我们构建以下程序来测试它们俩的取值和添值速度：

```java
import java.util.ArrayList;
import java.util.Arrays;
import java.util.LinkedList;
import java.util.List;

public class CollectionTest {

	private static final int N = 50000;
	private static List<Integer> val = null;

	static { // 初始化val
		Integer[] integers = new Integer[N];
		for (int i = 0; i < N; ++i) {
			integers[i] = (int) (Math.random() * 100);
		}
		val = Arrays.asList(integers);
	}

	public static Long getTime(List<Integer> list) { // 测试取值所消耗的时间
		Long start = System.currentTimeMillis(); // 开始时间
		for (int i = 0; i < N; ++i) {
			list.get(i);
		}
		Long end = System.currentTimeMillis(); // 结束时间
		return end - start;
	}

	public static Long addTime(List<Integer> list) { // 测试添加所消耗的时间
		Long start = System.currentTimeMillis(); // 开始时间
		for (int i = 0; i < N; ++i) {
			list.add(1, (int) (Math.random() * 100));
		}
		Long end = System.currentTimeMillis(); // 结束时间
		return end - start;
	}

	public static void main(String[] args) {
		System.out.println("ArrayList(get):" + getTime(new ArrayList<Integer>(val)) + "ms");
		System.out.println("LinkedList(get):" + getTime(new LinkedList<Integer>(val)) + "ms");

		System.out.println("ArrayList(add):" + addTime(new ArrayList<Integer>(val)) + "ms");
		System.out.println("LinkedList(add):" + addTime(new LinkedList<Integer>(val)) + "ms");
	}

}
/*结果：
	ArrayList(get):2ms
	LinkedList(get):1015ms
	ArrayList(add):896ms
	LinkedList(add):5ms
*/
```
由结果很容易得出结论：`ArrayList`取值比`LinkedList`取值快很多，`LinkedList`添值比`ArrayList`快很多。  
但是，如果添加值的index比较大，这个时候`LinkedList`就没有任何优势了，我们修改以上代码的`addTime`方法：
```java
public static Long addTime(List<Integer> list) { // 测试添加所消耗的时间
	Long start = System.currentTimeMillis(); // 开始时间
	for (int i = 0; i < N; ++i) {
		list.add(i, (int) (Math.random() * 100));  // 注意第一个参数变为i
	}
	Long end = System.currentTimeMillis(); // 结束时间
	return end - start;
}
/*结果：
ArrayList(add):566ms
LinkedList(add):1895ms
*/
```
结果`LinkedList`比`ArrayList`慢了很多，为什么呢，因为`LinkedList`要先找到`index`这个位置，一旦这个`index`过大，就比较费时了,这就是为什么上面结果会这么慢了。


## **使用集合需要注意的地方**

> * 由于Java集合存放的是对象的引用，循环使用`add()`方法需要注意，改变一个对象的值可能会改变集合中对象的值。


看如下一段代码，猜猜运行结果：
```java
import java.util.ArrayList;
import java.util.Collection;

public class Test {
	public static void main(String[] args) {
		Collection<Student> c = new ArrayList<>();
		Student s = new Student("hzj", 10);
		for(int i = 0; i < 10; ++i) {
			s.setAge(i);
			c.add(s);
		}
		c.forEach(System.out::println); // 结果？
	}
}

class Student {  // 实体类
	private String name;
	private int age;

	public Student() {
		super();
	}

	public Student(String name, int age) {
		super();
		this.name = name;
		this.age = age;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}

	@Override
	public String toString() {
		return "Student [name=" + name + ", age=" + age + "]";
	}

}
```

有没有人觉得会是如下结果：

```java
Student [name=hzj, age=0]
Student [name=hzj, age=1]
Student [name=hzj, age=2]
Student [name=hzj, age=3]
Student [name=hzj, age=4]
Student [name=hzj, age=5]
Student [name=hzj, age=6]
Student [name=hzj, age=7]
Student [name=hzj, age=8]
Student [name=hzj, age=9]
```

正确的结果其实是这样的：

```java
Student [name=hzj, age=9]
Student [name=hzj, age=9]
Student [name=hzj, age=9]
Student [name=hzj, age=9]
Student [name=hzj, age=9]
Student [name=hzj, age=9]
Student [name=hzj, age=9]
Student [name=hzj, age=9]
Student [name=hzj, age=9]
Student [name=hzj, age=9]
```

为什么是这个结果呢，因为`Java`集合存放的是对象的引用，并不是对象本身，所以当该`Student`对象更新之后，之前加入集合的对象的`Student`的值会因为该`Student`的对象更新而更新，但对象地址没有发生变化，所以当集合遍历的时候，由于是存放的地址，我们会取到同一个`Student`对象，而对象的值也更新成了最后一个循环所赋的值。  

我们可以用一个简单的例子证明一下(`Student`类使用上面的)：

```java
	Collection<Student> c = new ArrayList<>();
	Collection<Student> c1 = new ArrayList<>();
	Student s = new Student("hzj", 10);
	c.add(s);
	c1.add(s);
	System.out.println("修改前：");
	c.forEach(System.out::println);
	c1.forEach(System.out::println);
	
	for (Student student : c) {
		student.setAge(1000);
	}
	System.out.println("修改后：");
	c.forEach(System.out::println);
	c1.forEach(System.out::println);

	/*结果：

		修改前：
		Student [name=hzj, age=10]
		Student [name=hzj, age=10]
		修改后：
		Student [name=hzj, age=1000]
		Student [name=hzj, age=1000]
  	 */
```

修改`c`中的值，导致`c1`中也发生了变化，证明集合存放的是对象的引用，这个例子用的是`List`，其他集合也是一样的。  


> * 如果集合中存放的对象类型为不可变的（如`String`、`Long`、`Integer`等），则上面结论的不成立。

看一下代码：

```java
	Collection<Integer> c1 = new ArrayList<>();
	Collection<Integer> c2 = new ArrayList<>();
	Integer i1 = new Integer(130);
	c1.add(i1);
	c2.add(i1);
	c1.forEach(System.out::println);
	c2.forEach(System.out::println);
	for (Integer integer : c1) {
		integer = new Integer(100);
	}
	c1.forEach(System.out::println);
	c2.forEach(System.out::println);
```

结果如下，也就是之后的修改并没有影响到集合中的值，因为`Integer`是不可变类。

```java
	130
	130
	130
	130
```