---
layout: post
title: Java 中的类型传递问题解惑
---

我之前一直犯了一个错误，认为 Java 中是有引用传递的，其实不然，写这篇文章一方面是纠正自己的理解，另一个是希望看到文章的人不要犯同样的错。

# 类型传递有什么蹊跷？

要讨论 Java 中是值传递还是引用传递，先来看看如何定义值传递和引用传递。

- **值传递（pass by value）**：在调用函数时将实际参数拷贝一份传递到函数中，这样在函数中如果对参数进行修改，将不会影响到实际参数。
- **引用传递（pass by reference）**：在调用函数时将实际参数的地址直接传递到函数中，那么在函数中对参数所进行的修改，将影响到实际参数。

它们都是在发生函数调用时的一种参数求值策略，这是对调用函数时，求值和传值的方式的描述，而非传递的内容类型（内容指：是值类型还是引用类型，是值还是指针）。值类型/引用类型，是用于区分两种内存分配方式，值类型在栈上分配，引用类型在堆上分配。一个描述内存分配方式，一个描述参数求值策略，两者之间无任何依赖或约束关系。下面通过一些代码来理解这些话。

# Java 代码的味道

在 Java 中有基本类型如 `int`、`boolean`、`char`，也有对象类型如 `Object`、`HashMap`、`Integer` 等。

## 最简单的例子

```java
public static void swap(int a, int b) {
    int x = a;
    a = b;
    b = x;
    System.out.println("swap a: " + a);
    System.out.println("swap b: " + b);
}

public static void main(String[] args) {
    int a = 3;
    int b = 4;
    swap(a, b);
    System.out.println("a = " + a);
    System.out.println("b = " + b);
}
```

对于这段代码的输出是：

```shell
swap a: 4
swap b: 3
a = 3
b = 4
```

`int` 是一个基本类型，我们将 2 个变量的值传入到 `swap` 方法中，它们实际的结构是这样的：

<img src='{{ "/public/images/2018/11/pass_by_value_1.png" | prepend: site.cdnurl }}'  alt="iTerm2 设置" width="350px"/>

## 引用类型的例子

```java
static class Foo {
    String name;

    public Foo(String name) {
        this.name = name;
    }
}

static void change(Foo myfoo) {
    System.out.println("change input myfoo.name: " + myfoo.name);

    myfoo.name = "FiFi";
    System.out.println("change new   myfoo.name: " + myfoo.name);
}

public static void main(String[] args) {
    Foo foo = new Foo("Jerry");
    change(foo);

    System.out.println(foo.name);
}
```

对于这样一段代码的输出是：

```shell
change input myfoo.name: Jerry
change new   myfoo.name: FiFi
FiFi
```

对于 `change` 方法接收到的 `name` 是 Jerry 我们可以理解，修改后方法内输出的 FiFi 也可以理解，因为在同一函数体内，变量的修改必然是可见的。需要解释的就是外部的修改也会发生变化，这是为啥呢？

<img src='{{ "/public/images/2018/11/pass_by_value_2.png" | prepend: site.cdnurl }}'  alt="iTerm2 设置" width="600px"/>

看了这幅图你大概就明白了，我们 `new Foo("Jerry")` 这句话会在堆上开辟一块内存空间，假设叫它 `0xabcd` 好了，把这块内存赋值给一个名为 `foo` 的引用类型。进入 `change` 方法的时候就会传递一份 `foo` 引用类型的拷贝（参数入栈），我故意把这个变量名写为 `myfoo` 防止大家理解错误，`myfoo` 和 `foo` 这 2 个引用类型都指向同一块堆内存空间，在 `change` 方法内通过 `myfoo` 修改的值也会影响到其他指向这块内存的引用类型。

> 结论：对于引用类型，是通过传值的方式传引用（引用类型）。这句话可能会有点儿绕，不要死记硬背它。

上面的例子都是演示了 Java 中的值传递，假设 Java 里有引用传递会是什么样子呢？来看看这段代码

```java
static void change(Foo myfoo) {
    System.out.println("change input myfoo.name: " + myfoo.name);

    // 改变 foo 的指向，让它指向一个新的内存
    myfoo = new Foo("biezhi");
    System.out.println("change new   myfoo.name: " + myfoo.name);
}

public static void main(String[] args) {
    Foo foo = new Foo("Jerry");
    change(foo);

    System.out.println(foo.name);
}
```

这段代码的输出是

```shell
change input myfoo.name: Jerry
change new   myfoo.name: biezhi
Jerry
```

可能你看到这里已经恍然大悟了，如果 Java 中可以传递引用意味着 `myfoo = new Foo("biezhi");` 这行代码的修改会影响到之前的 `foo` 变量，会让 `foo` 和 `myfoo` 指向同一块新的地址，然后导致外部的 `foo.name` 会输出 biezhi。

我们看到结果不是这样的，正因为是传值（拷贝了一份）。这行代码会创建一个新对象，让 `myfoo` 指向新内存，而 `foo` 指向的还是旧的内存，所以 `myfoo` 的是不会影响到外部的。

## biezhi 的新疑问

看到上面其实值传递的概念和理解已经比较清晰了，之前又遇到一个问题，来看看代码吧

```java
static void change(Integer num) {
    System.out.println("change input num: " + num);
    num = 2333;
    System.out.println("change new   num: " + num);
}

public static void main(String[] args) {
    Integer num = 1;

    change(num);
    System.out.println(num);
}
```

我当时的疑问，`Integer` 是引用类型吗？传递给 `change` 方法的参数修改后会影响外部吗？

- `Integer` 创建出来的是一个对象（自动装箱），所以是引用类型
- 不会影响到外部

第二个问题为什么就不会影响到外部呢？原因是这样的，`Integer` 这个类（还有 `String`、`Double`、`Long` 等）比较特殊，它是 `final` 的，是不可变的。当我们使用 `num = 2333;` 这行代码去赋值的时候发生了什么？

这里我就不画图描述了，这行代码意味着给 `num` 又创建一个新的对象，所以方法内的 `num` 引用类型指向了 2333 所处的内存。当我们使用 `=` 进行操作时候修改的是引用，而不是一个单纯的赋值操作，相当于前面例子中的 `myfoo = new Foo("biezhi")`。如果你想实现 `Integer` 数值的修改可以试试 `AtomicInteger` 这个类。

# 小结

- 一个方法不能修改一个基本数据类型的参数
- 一个方法可以修改一个对象参数的状态
- 一个方法不能实现让对象参数引用一个新对象

之前对 Java 中值传递理解不够的原因是没有分清 **引用** 和 **引用传递** 是什么，只是在 Java 中理解起来有点绕，通过上面的小示例分析了这些问题出现的原因，如果你还有些模糊也可以参考我下方给出的链接学习，有什么疑问可以留言告诉我。

# 参考

- [Is Java “pass-by-reference” or “pass-by-value”?](https://stackoverflow.com/questions/40480/is-java-pass-by-reference-or-pass-by-value){:target="_blank"}
- [Java Method Arguments](http://www.cs.toronto.edu/~reid/web/javaparams.html){:target="_blank"}
- [Java 到底是值传递还是引用传递？](https://www.zhihu.com/question/31203609){:target="_blank"}