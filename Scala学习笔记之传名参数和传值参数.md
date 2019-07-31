# `Scala`传名参数和传值参数

[toc]

---

## `1.` 定义

`Scala`的解释器在解析函数参数`(function arguments)`时有两种方式：
> * 先计算参数表达式的值`(reduce the arguments)`，再应用到函数内部；或者是将未计算的参数表达式直接应用到函数内部。
> * 前者叫做传值调用`（call-by-value）`，后者叫做传名调用`（call-by-name）`。

例如：
```scala
object Add {  
    def addByName(a: Int, b: => Int) = a + b   
    def addByValue(a: Int, b: Int) = a + b   
}
```
`addByName`是传名调用，`addByValue`是传值调用。语法上可以看出，使用传名调用时，在参数名称和参数类型中间有一个" `=>`"符号。
以`a`为`2`，`b`为`2+2`为例，他们在`Scala`解释器进行参数规约`（reduction）`时的顺序分别是这样的：
```scala
    addByName(2, 2 + 2)  
->2 + (2 + 2)  
->2 + 4  
->6  
    
    addByValue(2, 2 + 2)  
->addByValue(2, 4)  
->2 + 4  
->6
```
可以看出，在进入函数内部前，传值调用方式就已经将参数表达式的值计算完毕，而传名调用是在函数内部进行参数表达式的值计算的。
这就造成了一种现象，每次使用传名调用时，解释器都会计算一次表达式的值。对于有副作用`(side-effect)`的参数来说，这无疑造成了两种调用方式结果的不同。

## `2.` 两者的比较

> * 传值调用在进入函数体之前就对参数表达式进行了计算，这避免了函数内部多次使用参数时重复计算其值，在一定程度上提高了效率。
> * 传名调用的一个优势在于，如果参数在函数体内部没有被使用到，那么它就不用计算参数表达式的值了。在这种情况下，传名调用的效率会高一点。

以一个具体的例子来说明传名参数的用法：
```scala
    var assertionsEnabled = true
    def myAssert(predicate: => Boolean ) = {
        if(assertionsEnabled && !predicate)
        throw new AssertionError
    }
```
这个`myAssert`函数的参数为一个函数类型，如果标志`assertionsEnabled`为`True`时，`mymyAssert` 根据`predicate` 的真假决定是否抛出异常，如果`assertionsEnabled` 为`false`,则这个函数什么也不做。

现在可以直接使用下面的语法来调用`myNameAssert`：
```scala
    myNameAssert(5 > 3)
```
看到这里，你可能会想我为什么不直接把参数类型定义为`Boolean`，比如：
```scala
    def boolAssert(predicate: Boolean ) = {
        if(assertionsEnabled && !predicate)
        throw new AssertionError
    }
```
它的调用也可以使用`boolAssert(5 > 3)`，和`myNameAssert` 调用看起来也没什么区别，其实两者有着本质的区别，一个是传值参数，一个是传名参数，在调用`boolAssert(5 > 3)`时，`5 > 3`是已经计算出为`true`，然后传递给`boolAssert`方法，而`myNameAssert(5 > 3)`，表达式`5 > 3`没有事先计算好传递给`myNameAssert`，而是先创建一个函数类型的参数值，这个函数的`apply`方法将计算`5 > 3`，然后这个函数类型的值作为参数传给`myNameAssert`。

因此这两个函数一个明显的区别是，如果设置`assertionsEnabled` 为`false`，然后试图计算 `x / 0 == 0`：
```scala
    var assertionsEnabled = false
    val x = 5
    boolAssert( x / 0 == 0) // 报错 java.lang.ArithmeticException: / by zero
    myNameAssert ( x / 0 == 0) // 不报错，因为没有调用predicate，则x / 0 == 0 没有执行

    def myAssert(predicate: => Boolean ) = {
        if(assertionsEnabled && !predicate)
        throw new AssertionError
    }

    def boolAssert(predicate: Boolean ) = {
        if(assertionsEnabled && !predicate)
        throw new AssertionError
    }
```
可以看到`boolAssert` 抛出 `java.lang.ArithmeticException: / by zero` 异常，这是因为这是个传值参数，首先计算 `x / 0` ，而抛出异常，而 `myNameAssert` 没有任何显示，这是因为这是个传名参数，传入的是一个函数类型的值，不会先计算`x / 0 == 0`,而在`myNameAssert` 函数体内，由于`assertionsEnabled`为`false`，传入的`predicate`没有必要计算(短路计算），因此什么也不会打印。如果我们把`myNameAssert` 修改下，把`predicate`放在前面：
```scala
    def myAssert(predicate: => Boolean ) = {
        if(!predicate && assertionsEnabled )
        throw new AssertionError
    }
    var assertionsEnabled = false
    val x = 5
    boolAssert( x / 0 == 0) // 报错 java.lang.ArithmeticException: / by zero
    myNameAssert ( x / 0 == 0) // 报错 java.lang.ArithmeticException: / by zero
```
则两个都会报错，因为`if`里面会先判断`predicate`。

## `3.` 自定义`while`循环

我们使用的while循环结构如下：
```scala
    while(/*条件*/){
      // 代码块
    }
```
我们可以把它看做为一个有两个参数的函数，即`while(flag:Boolean)(block: => Unit)`，具体实现如下：
```scala
object note {
  def main(args: Array[String]): Unit = {
    var i = 0;

    mywhile(i < 10) {
      println(i)
      i += 1
    }
    /* 上面那个循环等价于：

    mywhile(i < 10)(f)
    def f() = {
      println(i)
      i += 1
    }
    */

    def mywhile(flag: Boolean)(block: => Unit): Unit = {
      if (flag) { // 如果flag为真则执行代码块（代码块就是一个函数），并递归调用自己，实现循环的功能
        block
        mywhile(flag)(block)
      } // 如果flag为假，则不执行代码块，并停止循环，即停止递归
    }
  }
}
```
我们希望它打印`0`到`9`，但是事实并不如此，因为传`flag`参数的时候是用的传值调用，即一开始值就算完了是`（0 < 10 ： true），`，这会导致递归会一直进行下去，解决这个问题很简单，把`flag`改成传名调用就行了，如下：
```scala
object note {
  def main(args: Array[String]): Unit = {
    var i = 0;
    mywhile(i < 10) {
      println(i)
      i += 1
    }

    def mywhile(flag: => Boolean)(block: => Unit): Unit = {
      if (flag) {
        block
        mywhile(flag)(block)
      }
    }
  }
}
```