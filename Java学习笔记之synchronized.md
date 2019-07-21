# `Java学习笔记——synchronized关键字`


[toc]


---


## `synchronized`的三种应用方式

> * 修饰实例方法，作用于当前实例加锁，进入同步代码前要获得当前实例的锁。
> * 修饰静态方法，作用于当前类对象加锁，进入同步代码前要获得当前类对象的锁。
> * 修饰代码块，指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。

### （一）、`synchronized`作用于实例方法

用`synchronized`作用于实例方法，代表进入该方法前要获取到当前实例的锁。所谓的实例对象锁就是用`synchronized`修饰实例对象中的实例方法，注意是实例方法不包括静态方法，如下：

```java

public class SynchronizedTest {
	
	private int index = 0;

	public static void main(String[] args) {
		SynchronizedTest sync = new SynchronizedTest();
		new Thread(() -> {sync.test();}, "A").start();  // 获取了sync实例锁
		new Thread(() -> {sync.test();}, "B").start();  // 等待A线程释放sync锁
	}

	public synchronized void test() {
		for(int i = 0 ; i< 5; ++i) {
            try {
				Thread.sleep(100); // 为了增加效果
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println(Thread.currentThread().getName() + "\t index="+(++index));
		}
	}
}

/*
结果：
A	 index=1
A	 index=2
A	 index=3
A	 index=4
A	 index=5
B	 index=6
B	 index=7
B	 index=8
B	 index=9
B	 index=10
*/

```

从结果中可以看出，第一个线程（`A`线程）调用`test()`方法时获取了`sync`实例的锁，当第二个线程（`B`线程）调用`test()`方法时，会等待第一个线程释放`sync`锁，否则一直在阻塞状态。 

不难理解，如果一个线程访问了一个实例的`synchronized`非静态方法，其它的线程则无法访问该实例的其它任意`synchronized`非静态方法，毕竟一个对象只有一把锁。示例如下：

```java

public class SynchronizedTest {

	private int index = 0;

	public static void main(String[] args) {
		SynchronizedTest sync = new SynchronizedTest();
		new Thread(() -> {sync.test1();}, "A").start();
		new Thread(() -> {sync.test2();}, "B").start(); // A线程还未执行完，获取不到sync实例的锁，故B要等待A执行完才能执行test2()
	}

	public synchronized void test1() {
		for (int i = 0; i < 5; ++i) {
            try {
				Thread.sleep(100); // 为了增加效果
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println(Thread.currentThread().getName() + "\t index=" + (++index));
		}
	}

	public synchronized void test2() {
		for (int i = 0; i < 5; ++i) {
            try {
				Thread.sleep(100); // 为了增加效果
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println(Thread.currentThread().getName() + "\t index=" + (++index));
		}
	}
}
/*
结果：
A	 index=1
A	 index=2
A	 index=3
A	 index=4
A	 index=5
B	 index=6
B	 index=7
B	 index=8
B	 index=9
B	 index=10
*/
```
上述例子易知，如果一个线程访问了一个实例的`synchronized`非静态方法，其它的线程则无法访问该实例的其它任意`synchronized`非静态方法，即其它线程获取不到该实例的锁。  

如果是多个实例就不干扰，示例：

```java
public class SynchronizedTest {

	private int index = 0;

	public static void main(String[] args) {
		new Thread(() -> {new SynchronizedTest().test1();}, "A").start(); // new SynchronizedTest()对象
		new Thread(() -> {new SynchronizedTest().test2();}, "B").start(); // 又new一个，不需等待A释放锁
	}

	public synchronized void test1() {
		for (int i = 0; i < 5; ++i) {
            try {
				Thread.sleep(100); // 为了增加效果
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println(Thread.currentThread().getName() + "\t index=" + (++index));
		}
	}

	public synchronized void test2() {
		for (int i = 0; i < 5; ++i) {
            try {
				Thread.sleep(100); // 为了增加效果
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println(Thread.currentThread().getName() + "\t index=" + (++index));
		}
	}
}
/*
结果：
A	 index=1
A	 index=2
A	 index=3
B	 index=1
A	 index=4
A	 index=5
B	 index=2
B	 index=3
B	 index=4
B	 index=5
*/
```

### （二）、`synchronized`作用于静态方法

当`synchronized`作用于静态方法时，其锁就是当前类的`class`对象锁。由于静态成员不专属于任何一个实例对象，是类成员，因此通过`class`对象锁可以控制静态 成员的并发操作。 

需要注意的是如果一个线程`A`调用一个实例对象的非`static synchronized`方法，而线程`B`需要调用这个实例对象所属类的静态 `synchronized`方法，是允许的，不会发生互斥现象，因为访问静态 `synchronized` 方法占用的锁是当前类的`class`对象，而访问非静态 `synchronized` 方法占用的锁是当前实例对象锁。示例：


```java
public class SynchronizedTest {

	private static int index = 0;

	public static void main(String[] args) {
		SynchronizedTest sync = new SynchronizedTest();
		new Thread(() -> {sync.test1();}, "A").start();
		new Thread(() -> {sync.test2();}, "B").start();
	}

	public static synchronized void test1() {
		for (int i = 0; i < 5; ++i) {
			try {
				Thread.sleep(100); // 为了增加效果
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println(Thread.currentThread().getName() + "\t index = " + (++index));
		}
	}

	public synchronized void test2() {
		for (int i = 0; i < 5; ++i) {
			try {
				Thread.sleep(100); // 为了增加效果
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println(Thread.currentThread().getName() + "\t index = " + (++index));
		}
	}
}
/*
结果： B并没有等待A释放锁
A	 index = 1
B	 index = 2
B	 index = 4
A	 index = 3
A	 index = 5
B	 index = 6
A	 index = 7
B	 index = 8
A	 index = 9
B	 index = 10
*/
```

### （三）、`synchronized`作用于代码块
除了使用关键字修饰实例方法和静态方法外，还可以使用同步代码块，在某些情况下，我们编写的方法体可能比较大，同时存在一些比较耗时的操作，而需要同步的代码又只有一小部分，如果直接对整个方法进行同步操作，可能会得不偿失，此时我们可以使用同步代码块的方式对需要同步的代码进行包裹，这样就无需对整个方法进行同步操作了，同步代码块的使用示例如下：

**1. 锁当前实例对象：**
```java
public class SynchronizedTest {

	private int index = 0;

	public static void main(String[] args) throws InterruptedException {
		SynchronizedTest sync = new SynchronizedTest();
		new Thread(() -> {sync.test();}, "A").start();
		new Thread(() -> {sync.test();}, "B").start();
		
		Thread.sleep(2000);
		System.out.println("=============");
		
		new Thread(() -> {new SynchronizedTest().test();}, "C").start();
		new Thread(() -> {new SynchronizedTest().test();}, "D").start();
	}

	public void test() {
		synchronized (this) {
			for (int i = 0; i < 5; ++i) {
				try {
					Thread.sleep(100); // 为了增加效果
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				System.out.println(Thread.currentThread().getName() + "\t index=" + (++index));
			}
		}
	}
}
/*
结果： A和B共用一个实例，所以B要等A释放锁
       C和D不是共用一个实例，故D不需要等B释放锁
A	 index=1
A	 index=2
A	 index=3
A	 index=4
A	 index=5
B	 index=6
B	 index=7
B	 index=8
B	 index=9
B	 index=10
=============
D	 index=1
C	 index=1
C	 index=2
D	 index=2
D	 index=3
C	 index=3
C	 index=4
D	 index=4
C	 index=5
D	 index=5
*/
```

**2. 锁`class`对象：**

```java
public class SynchronizedTest {

	private int index = 0;

	public static void main(String[] args) throws InterruptedException {
		SynchronizedTest sync = new SynchronizedTest();
		new Thread(() -> {sync.test();}, "A").start();
		new Thread(() -> {sync.test();}, "B").start();
		
		Thread.sleep(2000);
		System.out.println("=============");
		
		new Thread(() -> {new SynchronizedTest().test();}, "C").start();
		new Thread(() -> {new SynchronizedTest().test();}, "D").start();
	}


	public void test() {
		synchronized (SynchronizedTest.class) { // 锁class对象
			for (int i = 0; i < 5; ++i) {
				try {
					Thread.sleep(100); // 为了增加效果
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				System.out.println(Thread.currentThread().getName() + "\t index=" + (++index));
			}
		}
	}
}
/*
结果： ABCD都是SynchronizedTest类的对象，故都要互斥访问，B等A，D等C
A	 index=1
A	 index=2
A	 index=3
A	 index=4
A	 index=5
B	 index=6
B	 index=7
B	 index=8
B	 index=9
B	 index=10
=============
C	 index=1
C	 index=2
C	 index=3
C	 index=4
C	 index=5
D	 index=1
D	 index=2
D	 index=3
D	 index=4
D	 index=5
*/
```


## `synchronized`的应用例子——单例模式

单例模式一般也是将`synchronized`作用于代码块，单例模式代码：

```java
/**
 * 单例创建的方法
 * 1、懒汉式
 * 1)、构造器私有化
 * 2)、声明私有的静态属性
 * 3)、提供对外访问属性的静态方法，确保该对象存在
 * @author Au
 *
 */
public class MyJVM {
	private static MyJVM instance;
	private MyJVM() {
		
	}
	public static MyJVM getInstance() {
		if(null == instance) { //提供效率
			synchronized (MyJVM.class) {
				if(null == instance) { //安全
					instance = new MyJVM();
				}
			}
		}
		return instance;
	}
}
```