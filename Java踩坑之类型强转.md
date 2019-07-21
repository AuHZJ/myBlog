# Java踩坑系列——类型强转（Object[] 转 String[]）

[toc]

---

最近开发中遇到了一个`bug`，就是`List`的`toArray`方法，他返回的是`Object[]`类型，当我们将它转换为`String[]`类型的时候并不会提示错误，但运行时就会报如下错，其实就是类型转换异常：

```java
    // 运行代码：
    List<String> list = new ArrayList<>();
    Object[] objects = list.toArray();
    String[] strs = (String[]) objects;

    // 会报如下错：
    Exception in thread "main" java.lang.ClassCastException: 
                            [Ljava.lang.Object; cannot be cast to [Ljava.lang.String;
```

解决办法就是调用`List`的另一个方法：`T[] toArray(T[] a)`，即传什么类型的参数就返回什么类型的参数，我们可以这样调用：

```java
    List<String> list = new ArrayList<>();
    Object[] objects = list.toArray();
    String[] strs = list.toArray(new String[0]);
```

那么为什么第一个会报类型转换异常呢，原因就是在`Java`的类型强转中，**被强转的对象所指对象类型 必须是 需要强转类型的同类型或者子类型，不能是父类型**，比如：

```java
    Object obj = new Object(); 
    String str = (String) obj;
```

这样是行不通的，会报类型转换异常，在这里，被强转对象是`obj`，是`Object`类型的，它指向的也是`Object`类型的，而它被要求转为`String`类型，`Object`类型并不是`String`的子类型，而是`String`的父类型，所以会报类型转换异常。

那么对于第一个例子中的错误就很好理解了，`list.toArray()`返回的是`Object[]`类型，`objects`指向了它，而后面要求将`objects`转换为`String[]`类型，`Object[]`并不是`String[]`的同类型或子类型，如下代码是行得通的，因为objs指向的类型是`String[]`类型：

```java
    Object[] objs = new String[10];
    String[] strs = (String[]) objs;
```