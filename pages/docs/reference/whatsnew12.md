---
type: doc
layout: reference
title: "Kotlin 1.2 的新特性"
---

# Kotlin 1.2 的新特性

## 目录

* [多平台项目](#多平台项目实验性的)
* [其他语言特性](#其他语言特性)
* [标准库](#标准库)
* [JVM 后端](#jvm-后端)
* [JavaScript 后端](#javascript-后端)

{:#多平台项目实验性的}

## 多平台项目（实验性的）

多平台项目是 Kotlin 1.2 中的一个新的**实验性的**功能，允许你在支持 Kotlin 的目标平台——JVM、JavaScript 以及（将来的）Native 之间重用代码。在多平台项目中，你有三种模块：

* 一个*公共*模块包含平台无关代码，以及无实现的依赖平台的 API 声明。
* *平台*模块包含通用模块中的平台相关声明在指定平台的实现，以及其他平台相关代码。
* 常规模块针对指定的平台，既可以是平台模块的依赖，也可以依赖平台模块。

当你为指定平台编译多平台项目时，既会生成公共代码也会生成平台相关代码。

多平台项目支持的一个主要特点是可以通过**预期声明与实际声明**来表达公共代码对平台相关部分的依赖关系。一个预期声明指定一个 API（类、接口、注解、顶层声明等）。一个实际声明要么是该 API 的平台相关实现，要么是一个引用到在一个外部库中该 API 的一个既有实现的别名。这是一个示例：

在公共代码中：

```kotlin
// 预期平台相关 API:
expect fun hello(world: String): String

fun greet() {
    // 该预期 API 的用法：
    val greeting = hello("multi-platform world")
    println(greeting)
}

expect class URL(spec: String) {
    open fun getHost(): String
    open fun getPath(): String
}
```

在 JVM 平台代码中：

```kotlin
actual fun hello(world: String): String =
    "Hello, $world, on the JVM platform!"

// 使用既有平台相关实现：
actual typealias URL = java.net.URL
```

关于构建多平台项目的详细信息与步骤，请参见其[documentation](http://kotlinlang.org/docs/reference/multiplatform.html)<!--
-->。

## 其他语言特性

### 注解中的数组字面值

自 Kotlin 1.2 起，注解的数组参数可以通过新的数组字面值语法传入，而无需<!--
-->使用 `arrayOf` 函数：

```kotlin
@CacheConfig(cacheNames = ["books", "default"])
public class BookRepositoryImpl {
    // ……
}
```

该数组字面值语法仅限于注解参数。

### lateinit 顶层属性与局部变量

`lateinit` 修饰符现在可以用于顶层属性与局部变量了。例如，后者可用于<!--
-->当一个 lambda 表达式作为构造函数参数传给一个对象时，引用另一个<!--
-->必须稍后定义的对象：

<div class="sample" markdown="1" data-min-compiler-version="1.2">

```kotlin
class Node<T>(val value: T, val next: () -> Node<T>)

fun main(args: Array<String>) {
    //sampleStart
    // A cycle of three nodes:
    lateinit var third: Node<Int>
    
    val second = Node(2, next = { third })
    val first = Node(1, next = { second })
    
    third = Node(3, next = { first })
    //sampleEnd
    
    val nodes = generateSequence(first) { it.next() }
    println("Values in the cycle: ${nodes.take(7).joinToString { it.value.toString() }}, ...")
}
```
</div>

### 检查 lateinit 变量是否已初始化

现在可以通过属性引用的 `isInitialized` 来检测该 lateinit var 是否已初始化：

<div class="sample" markdown="1" data-min-compiler-version="1.2">

```kotlin
class Foo {
    lateinit var lateinitVar: String
    
    fun initializationLogic() {
        //sampleStart
        println("isInitialized before assignment: " + this::lateinitVar.isInitialized)
        lateinitVar = "value"
        println("isInitialized after assignment: " + this::lateinitVar.isInitialized)    
        //sampleEnd
    }
}

fun main(args: Array<String>) {
	Foo().initializationLogic()
}
```
</div>

### 内联函数带有默认函数式参数

内联函数现在允许其内联函式数参数具有默认值：

<div class="sample" markdown="1" data-min-compiler-version="1.2">

```kotlin
//sampleStart
inline fun <E> Iterable<E>.strings(transform: (E) -> String = { it.toString() }) = 
    map { transform(it) }

val defaultStrings = listOf(1, 2, 3).strings()
val customStrings = listOf(1, 2, 3).strings { "($it)" } 
//sampleEnd

fun main(args: Array<String>) {
    println("defaultStrings = $defaultStrings")
    println("customStrings = $customStrings")
}
```
</div>

### 源自显式类型转换的信息会用于类型推断

Kotlin 编译器现在可将类型转换信息用于类型推断。如果你调用一个<!--
-->返回类型参数 `T` 的泛型方法并将返回值转换为指定类型 `Foo`，那么编译器现在知道<!--
-->对于本次调用需要绑定类型为 `Foo`。

这对于 Android 开发者来说尤为重要，因为编译器现在可以正确分析
Android API 级别 26 中的泛型 `findViewById` 调用：

```kotlin
val button = findViewById(R.id.button) as Button
```

### 智能类型转换改进

当一个变量有安全调用表达式与空检测赋值时，其智能转换现在也可以应用于<!--
-->安全调用接收者：

<div class="sample" markdown="1" data-min-compiler-version="1.2">

```kotlin
fun countFirst(s: Any): Int {
    //sampleStart
    val firstChar = (s as? CharSequence)?.firstOrNull()
    if (firstChar != null)
       return s.count { it == firstChar } // s: Any 会智能转换为 CharSequence
    
    val firstItem = (s as? Iterable<*>)?.firstOrNull()
    if (firstItem != null)
       return s.count { it == firstItem } // s: Any 会智能转换为 Iterable<*>
    //sampleEnd
    
    return -1
}

fun main(args: Array<String>) {
    val string = "abacaba"
    val countInString = countFirst(string)
    println("called on \"$string\": $countInString")
    
    val list = listOf(1, 2, 3, 1, 2)
    val countInList = countFirst(list)
    println("called on $list: $countInList")
}
```
</div>

智能转换现在也允许用于在 lambda 表达式中局部变量，只要这些局部变量仅在 lambda 表达式之前修改即可：

<div class="sample" markdown="1" data-min-compiler-version="1.2">

```kotlin
fun main(args: Array<String>) {
    val flag = args.size == 0
    
    //sampleStart
    var x: String? = null
    if (flag) x = "Yahoo!"

    run {
        if (x != null) {
            println(x.length) // x 会智能转换为 String
        }
    }
    //sampleEnd
}
```
</div>

### 支持 ::foo 作为 this::foo 的简写

现在写绑定到 `this` 成员的可调用引用可以无需显式接收者，即 `::foo` 取代
`this:: foo`。这也使在引用外部接收者的成员的 lambda 表达式中使用可调用引用更加方便<!--
-->。

### 阻断性改动：try 块后可靠智能转换

Kotlin 以前将 `try` 块中的赋值语句用于块后的智能转换，这可能会破坏类型安全与空安全<!--
-->并引发运行时故障。这个版本修复了该问题，使智能转换更加严格，但可能会破坏一些<!--
-->依靠这种智能转换的代码。

如果要切换到旧版智能转换行为，请传入回退标志 `-Xlegacy-smart-cast-after-try` 作为编译器<!--
-->参数。该参数会在 Kotlin 1.3 中弃用。

### 弃用：数据类弃用 copy

当从已具有签名相同的 `copy` 函数的类型派生数据类时，为数据类生成的 `copy`
实现使用超类型的默认值，这导致反直觉行为，
或者导致运行时失败，如果超类型中没有默认参数的话。

导致 `copy` 冲突的继承在 Kotlin 1.2 中已弃用并带有警告，
而在 Kotlin 1.3 中将会是错误。

### 弃用：枚举条目中的嵌套类型

由于初始化逻辑的问题，已弃用在枚举条目内部定义一个非 `inner class`
的嵌套类。这在 Kotlin 1.2 中会引起警告，而在 Kotlin 1.3 中会成为错误。

### 弃用：vararg 单个命名参数

为了与注解中的数组字面值保持一致，向一个命名参数形式的 vararg 参数传入单个项目<!--
-->的用法（`foo(items = i)`）已被弃用。请使用伸展操作符连同相应的<!--
-->数组工厂函数：

```kotlin
foo(items = *intArrayOf(1))
```

在这种情况下有一项防止性能下降的优化可以消除冗余的数组创建。
单参数形式在 Kotlin 1.2 中会产生警告，而在 Kotlin 1.3 中会放弃。

### 弃用：扩展 Throwable 的泛型类的内部类

继承自 `Throwable` 的泛型类的内部类可能会在 throw-catch 场景中违反类型安全性，因此<!--
-->已弃用，在 Kotlin 1.2 中会是警告，而在 Kotlin 1.3 中会是错误。

### 弃用：修改只读属性的幕后字段

通过在自定义 getter 中赋值 `field = ……` 来修改只读属性的幕后字段的用法已被弃用，
在 Kotlin 1.2 中会是警告，而在 Kotlin 1.3 中会是错误。

## 标准库

### Kotlin standard library artifacts and split packages

The Kotlin standard library is now fully compatible with the Java 9 module system, which forbids split packages 
(multiple jar files declaring classes in the same package). In order to support that, new artifacts `kotlin-stdlib-jdk7` 
and `kotlin-stdlib-jdk8` are introduced, which replace the old `kotlin-stdlib-jre7` and `kotlin-stdlib-jre8`.
 
The declarations in the new artifacts are visible under the same package names from the Kotlin point of view, but have 
different package names for Java. Therefore, switching to the new artifacts will not require any changes to 
your source code.

Another change made to ensure compatibility with the new module system is removing the deprecated 
declarations in the `kotlin.reflect` package from the `kotlin-reflect` library. If you were using them, you need 
to switch to using the declarations in the `kotlin.reflect.full` package, which is supported since Kotlin 1.1.

### windowed, chunked, zipWithNext

New extensions for `Iterable<T>`, `Sequence<T>`, and `CharSequence` cover such use cases as buffering or 
batch processing (`chunked`), sliding window and computing sliding average (`windowed`) , and processing pairs 
of subsequent items (`zipWithNext`):

<div class="sample" markdown="1" data-min-compiler-version="1.2">

```kotlin
fun main(args: Array<String>) {
    //sampleStart
    val items = (1..9).map { it * it }

    val chunkedIntoLists = items.chunked(4)
    val points3d = items.chunked(3) { (x, y, z) -> Triple(x, y, z) }
    val windowed = items.windowed(4)
    val slidingAverage = items.windowed(4) { it.average() }
    val pairwiseDifferences = items.zipWithNext { a, b -> b - a }
    //sampleEnd
    
    println("items: $items\n")
    
    println("chunked into lists: $chunkedIntoLists")
    println("3D points: $points3d")
    println("windowed by 4: $windowed")
    println("sliding average by 4: $slidingAverage")
    println("pairwise differences: $pairwiseDifferences")
}
```
</div>

### fill, replaceAll, shuffle/shuffled

A set of extension functions was added for manipulating lists: `fill`, `replaceAll` and `shuffle` for `MutableList`, 
and `shuffled` for read-only `List`:

<div class="sample" markdown="1" data-min-compiler-version="1.2">

```kotlin
fun main(args: Array<String>) {
    //sampleStart
    val items = (1..5).toMutableList()
    
    items.shuffle()
    println("Shuffled items: $items")
    
    items.replaceAll { it * 2 }
    println("Items doubled: $items")
    
    items.fill(5)
    println("Items filled with 5: $items")
    //sampleEnd
}
```
</div>

### Math operations in kotlin-stdlib

Satisfying the longstanding request, Kotlin 1.2 adds the `kotlin.math` API for math operations that is common 
for JVM and JS and contains the following:

* Constants: `PI` and `E`;
* Trigonometric: `cos`, `sin`, `tan` and inverse of them: `acos`, `asin`, `atan`, `atan2`;
* Hyperbolic: `cosh`, `sinh`, `tanh` and their inverse: `acosh`, `asinh`, `atanh`
* Exponentation: `pow` (an extension function), `sqrt`, `hypot`, `exp`, `expm1`;
* Logarithms: `log`, `log2`, `log10`, `ln`, `ln1p`;
* Rounding:
    * `ceil`, `floor`, `truncate`, `round` (half to even) functions;
    * `roundToInt`, `roundToLong` (half to integer) extension functions;
* Sign and absolute value:
    * `abs` and `sign` functions;
    * `absoluteValue` and `sign` extension properties;
    * `withSign` extension function;
* `max` and `min` of two values;
* Binary representation:
    * `ulp` extension property;
    * `nextUp`, `nextDown`, `nextTowards` extension functions;
    * `toBits`, `toRawBits`, `Double.fromBits` (these are in the `kotlin` package).

The same set of functions (but without constants) is also available for `Float` arguments.

### Operators and conversions for BigInteger and BigDecimal

Kotlin 1.2 introduces a set of functions for operating with `BigInteger` and `BigDecimal` and creating them 
from other numeric types. These are:

* `toBigInteger` for `Int` and `Long`;
* `toBigDecimal` for `Int`, `Long`, `Float`, `Double`, and `BigInteger`;
* Arithmetic and bitwise operator functions:
    * Binary operators  `+`, `-`, `*`, `/`, `%` and infix functions `and`, `or`, `xor`, `shl`, `shr`;
    * Unary operators `-`, `++`, `--`, and a function `inv`.

### Floating point to bits conversions

New functions were added for converting `Double` and `Float` to and from their bit representations:

* `toBits` and `toRawBits` returning `Long` for `Double` and `Int` for `Float`;
* `Double.fromBits` and `Float.fromBits` for creating floating point numbers from the bit representation.

### Regex is now serializable

The `kotlin.text.Regex` class has become `Serializable` and can now be used in serializable hierarchies.

### Closeable.use calls Throwable.addSuppressed if available

The `Closeable.use` function calls `Throwable.addSuppressed` when an exception is thrown during closing the resource 
after some other exception. 

To enable this behavior you need to have `kotlin-stdlib-jdk7` in your dependencies.

## JVM 后端

### Constructor calls normalization

Ever since version 1.0, Kotlin supported expressions with complex control flow, such as try-catch expressions and 
inline function calls. Such code is valid according to the Java Virtual Machine specification. Unfortunately, some 
bytecode processing tools do not handle such code quite well when such expressions are present in the arguments 
of constructor calls.

To mitigate this problem for the users of such bytecode processing tools, we’ve added a command-line 
option (`-Xnormalize-constructor-calls=MODE`) that tells the compiler to generate more Java-like bytecode for such 
constructs. Here `MODE` is one of:

* `disable` (default) – generate bytecode in the same way as in Kotlin 1.0 and 1.1;
* `enable` – generate Java-like bytecode for constructor calls. This can change the order in which the classes are 
 loaded and initialized;
* `preserve-class-initialization` – generate Java-like bytecode for constructor calls, ensuring that the class 
initialization order is preserved. This can affect overall performance of your application; use it only if you have 
some complex state shared between multiple classes and updated on class initialization.

The “manual” workaround is to store the values of sub-expressions with control flow in variables, instead of 
evaluating them directly inside the call arguments. It’s similar to `-Xnormalize-constructor-calls=enable`.

### Java-default method calls 

Before Kotlin 1.2, interface members overriding Java-default methods while targeting JVM 1.6 produced a warning on 
super calls: `Super calls to Java default methods are deprecated in JVM target 1.6. Recompile with '-jvm-target 1.8'`. 
In Kotlin 1.2, there's an **error** instead, thus requiring any such code to be compiled with JVM target 1.8.

### Breaking change: consistent behavior of x.equals(null) for platform types

Calling `x.equals(null)` on a platform type that is mapped to a Java primitive 
(`Int!`, `Boolean!`, `Short`!, `Long!`, `Float!`, `Double!`, `Char!`) incorrectly returned `true` when `x` was null. 
Starting with Kotlin 1.2, calling `x.equals(...)` on a null value of a platform type **throws an NPE** 
(but `x == ...` does not).

To return to the pre-1.2 behavior, pass the flag `-Xno-exception-on-explicit-equals-for-boxed-null` to the compiler.

### Breaking change: fix for platform null escaping through an inlined extension receiver

Inline extension functions that were called on a null value of a platform type did not check the receiver for null and 
would thus allow null to escape into the other code. Kotlin 1.2 forces this check at the call sites, throwing an exception
if the receiver is null.

To switch to the old behavior, pass the fallback flag `-Xno-receiver-assertions` to the compiler.

## JavaScript 后端

### TypedArrays support enabled by default

The JS typed arrays support that translates Kotlin primitive arrays, such as `IntArray`, `DoubleArray`, 
into [JavaScript typed arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Typed_arrays), that was 
previously an opt-in feature, has been enabled by default.

## Tools

### Warnings as errors

The compiler now provides an option to treat all warnings as errors. Use `-Werror` on the command line, or the 
following Gradle snippet:

```groovy
compileKotlin {
    kotlinOptions.allWarningsAsErrors = true
}
```
