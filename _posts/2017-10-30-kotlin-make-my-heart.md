---
layout: post
title: 让我心动的Kotlin
date: 2017-10-30 23:16
tags: Kotlin
categories: 编程
---

[Kotlin](http://kotlinlang.org/)是一门务实、简洁和安全的语言，专注于互操作性。

*注：本文基于Kotlin 1.3编写。*

## Hello world!

```java
fun main() { // 1
    println("Hello world!") // 2
}
```

*1*

支持定义 *top-level* 函数，告别工具类。

`fun`关键字定义函数，参数声明格式是：`name: Type`。

`main`函数是程序执行的主入口。

*2*

`println`是内置*top-level*函数，打印消息到标准输出流并换行。

不需要`;`。

## 文档注释支持Markdown

```java
/**
 * 这是`文档注释`，它支持 **Markdown**。
 */
fun main() {
    println("Hello world!")
}
```

## 变量

`var`定义可变变量，`val`定义只读变量，初始化后不能再次被赋值。

```java
fun main() {
    var age: Int = 27
    val name: String = "Belin Wu"
}
```

## 类型推导

可以不写类型，编译器会自动推导。

```java
fun main() {
    var age = 27 // age is Int
    val name = "Belin Wu" // name is String
}
```

## 字符串模板

字符串里可以直接嵌套变量或表达式。

```java
fun main() {
    val name = "Kotlin"
    println("The length of $name is ${name.length}") // The length of Kotlin is 6
}
```

## 原始字符串

用三引号`"""`定义的字符串里可以包含任意字符，不需要转义。

```java
fun main() {
    val text = """
    for (c in "foo")
        print(c)
"""
    print(text)
}
```

## 没有`new`

直接调用构造函数就可以创建实例。

```java
fun main() {
    val version = KotlinVersion(1, 1, 51)
}
```

## 属性

不用定义`setter`和`getter`方法。

```java
class User(var name: String)

fun main() {
    val u = User("Belin")
    u.name = "Belin Wu"
    println("My name is ${u.name}")
}
```

## 定义对象

对象声明将类的声明和实例创建结合了起来，一步到位。`Singleton`同时也是一个单例。

```java
object Singleton

fun main() {
    println(Singleton) // Singleton@6e0be858
    println(Singleton::class) // class Singleton
}
```

## 类型检查与智能转换

判断对象是不是某个类型用`is`；判断对象不是某个类型用`!is`

```java
fun main() {
    val obj: Any = "Kotlin"
    println(obj is String) // true
    println(obj !is String) // false
}
```

确定对象是期望的类型后就可以直接调用它的属性或成员函数，不需要强制类型转换。

```java
fun main() {
    val obj: Any = "Kotlin"
    if (obj is String) {
        println(obj.length)
    }
}
```

## 类型别名

为已有类型定义别名有缩短类型名称、让名字更符合使用场景、简化范型或函数类型等好处。

```java
typealias Age = Byte
typealias Hobby = Set<String>
typealias Predicate<T> = (T) -> Boolean

class Url {
    inner class Builder
}

typealias UrlBuilder = Url.Builder
```

## 可空类型

Kotlin 类型分为：可空类型、不可空类型。`null`只允许将赋值给可空类型，否则会出现编译错误。

```java
fun main() {
    val str: String? = null
    val error: String = null // compile error
}
```

在类型的后面加上`?`就是该类型对应的可空类型。

调用可空类型的属性或成员函数有一种安全的操作符：`?.`，当调用者是`null`时，调用结果也为`null`。当在多个可空类型上做级联调用时可以省去嵌套`if`判断，代码更加简洁。

```java
fun main() {
    val str: String? = null
    str?.length?.toString()?.length / / null
}
```

## 默认参数值

有默认值的函数可以减少重载、困惑。

```java
fun joinToString(array: Array<String>, separator: String = ", "): String {
    return array.joinToString(separator)
}

fun main() {
    joinToString(args)
    joinToString(args, " ")
}
```

## 数据类

当使用`data class`定义数据类时，编译器会自动生成`toString()`，`equals()`，`hashCode()`，`copy()`等成员函数的字节码。

```java
data class User(val name: String, var age: Int)

fun main() {
    val user = User("Belin Wu", 27)
    user.equals(user) // true
    user.hashCode() // hash code
    user.copy(age = 28) // copy the user to new user and modify the age property
    println(user.toString()) // User(name=Belin Wu, age=27)
}
```

## 解构声明

将对象解构可以一次性声明并初始化多个变量。

```java
data class User(var name: String, var age: Int = 1)

fun main() {
    val u = User("Belin")
    val (name, age) = u
    print("name is $name, age is $age")
}
```

## 多返回值

使用解构声明可以实现返回多值的函数。

```java
fun main() {
    val (father, mother) = getParent()
    println(father.name)
    println(mother.name)
}

fun getParent(): Pair<Person, Person> {
    return Pair(Person("Father"), Person("Mother"))
}

class Person(var name: String)
```

`Pair<A, B>`是一个内置的数据类，它有`component1`和`component2`方法，Kotlin编译器会按顺序一一调用`componentN`方法并将结果分别赋值给`father`和`mother`变量。其中`component1`方法返回类声明的第一个属性，而`component2`方法返回第二个属性，依此类推。

举一反三，使用另一个内置的数据类`Triple<A, B, C>`就可以实现三个返回值的函数。

要一定是数据类吗？并不是，按照约定，在类中有定义N个`componentN`方法即可。

```java
class MyPair<out A, out B>(val a: A, val b: B) {
    operator fun component1() = a

    operator fun component2() = b
}
```

不再需要中间变量，代码更简洁。

## Delegation 机制

```kotlin
interface Base {
   fun print()
}

class BaseImpl(val x: Int) : Base {
   override fun print() { print(x) }
}

class Derived(b: Base) : Base by b

fun main() {
   val b = BaseImpl(10)
   Derived(b).print()
}
```

## 参考资料

- [Kotlin in Action](https://www.manning.com/books/kotlin-in-action)
- [Kotlin Reference](http://kotlinlang.org/docs/reference/)
