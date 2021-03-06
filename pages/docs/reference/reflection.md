---
type: doc
layout: reference
category: "Syntax"
title: "反射"
---

# 反射

反射是这样的一组语言和库功能，它允许在运行时自省你的程序的结构。
Kotlin 让语言中的函数和属性做为一等公民、并对其自省（即在运行时获悉<!--
-->一个名称或者一个属性或函数的类型）与简单地使用函数式或响应式风格紧密相关。

> On the JavaScript platform, only class references are currently supported. [Learn more about reflection in Kotlin/JS](js-reflection.html).
{:.note}

## JVM dependency

On the JVM platform, the runtime component required for using the reflection features is distributed as a separate
artifact `kotlin-reflect.jar` in the Kotlin compiler distribution. This is done to reduce the required size of the runtime
library for applications that do not use reflection features.

To use reflection in a Gradle or Maven project, add the dependency on `kotlin-reflect`:
* In Gradle:
  <div class="multi-language-sample" data-lang="groovy">
  <div class="sample" markdown="1" theme="idea" mode="groovy" data-highlight-only>
  
  ```groovy
  dependencies {
      implementation "org.jetbrains.kotlin:kotlin-reflect:{{ site.data.releases.latest.version }}"
  }
  ```
  
  </div>
  </div>
   
  <div class="multi-language-sample" data-lang="kotlin">
  <div class="sample" markdown="1" theme="idea" mode="kotlin" data-highlight-only>
  
  ```kotlin
  dependencies {
    implementation("org.jetbrains.kotlin:kotlin-reflect:{{ site.data.releases.latest.version }}")
  }
  ```
  
  </div>
  </div>
  
  * In Maven:

  <div class="sample" markdown="1" mode="xml" auto-indent="false" theme="idea" data-highlight-only>
    
  ```xml
  <dependencies>
      <dependency>
          <groupId>org.jetbrains.kotlin</groupId>
          <artifactId>kotlin-reflect</artifactId>
      </dependency>
  </dependencies>
  ```
  </div>

If you don't use Gradle or Maven, make sure you have `kotlin-reflect.jar` in the classpath of your project.
In other supported cases (IntelliJ IDEA projects, using command-line compiler or Ant),
it is added by default. In command-line compiler and Ant, you can use `-no-reflect` compiler option to exclude
`kotlin-reflect.jar` from the classpath. 

## 类引用

最基本的反射功能是获取 Kotlin 类的运行时引用。要获取对<!--
-->静态已知的 Kotlin 类的引用，可以使用 _类字面值_ 语法：

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
val c = MyClass::class
```
</div>

该引用是 [KClass](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-class/index.html) 类型的值。

请注意，Kotlin 类引用与 Java 类引用不同。要获得 Java 类引用，
请在 `KClass` 实例上使用 `.java` 属性。

{:#绑定的类引用自-11-起}

## 绑定的类引用（自 1.1 起）

通过使用对象作为接收者，可以用相同的 `::class` 语法获取指定对象的类的引用：

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
val widget: Widget = ……
assert(widget is GoodWidget) { "Bad widget: ${widget::class.qualifiedName}" }
```
</div>

你可以获取对象的精确类的引用，例如 `GoodWidget` 或 `BadWidget`，尽管接收者表达式的类型是 `Widget`。

## 可调用引用

函数、属性以及构造函数的引用，除了作为自省程序结构外，
还可以用于调用或者用作[函数类型](lambdas.html#函数类型)的实例。

所有可调用引用的公共超类型是 [`KCallable<out R>`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-callable/index.html)，
其中 `R` 是返回值类型，对于属性是属性类型，对于构造函数是所构造类型。

### 函数引用

当我们有一个具名函数声明如下：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun isOdd(x: Int) = x % 2 != 0
```
</div>

我们可以很容易地直接调用它（`isOdd(5)`），但是我们也可以将其作为一个函数类型的值，例如将其传给<!--
-->另一个函数。为此，我们使用 `::` 操作符：

<div class="sample" markdown="1" theme="idea">

```kotlin
fun isOdd(x: Int) = x % 2 != 0

fun main() {
//sampleStart
    val numbers = listOf(1, 2, 3)
    println(numbers.filter(::isOdd))
//sampleEnd
}
```
</div>

这里 `::isOdd` 是函数类型 `(Int) -> Boolean` 的一个值。

函数引用属于 [`KFunction<out R>`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-function/index.html)
的子类型之一，取决于参数个数，例如 `KFunction3<T1, T2, T3, R>`。

当上下文中已知函数期望的类型时，`::` 可以用于重载函数。
例如：

<div class="sample" markdown="1" theme="idea">

```kotlin
fun main() {
//sampleStart
    fun isOdd(x: Int) = x % 2 != 0
    fun isOdd(s: String) = s == "brillig" || s == "slithy" || s == "tove"
    
    val numbers = listOf(1, 2, 3)
    println(numbers.filter(::isOdd)) // 引用到 isOdd(x: Int)
//sampleEnd
}
```
</div>

或者，你可以通过将方法引用存储在具有显式指定类型的变量中来提供必要的上下文：

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
val predicate: (String) -> Boolean = ::isOdd   // 引用到 isOdd(x: String)
```
</div>

如果我们需要使用类的成员函数或扩展函数，它需要是限定的，例如 `String::toCharArray`。

请注意，即使以扩展函数的引用初始化一个变量，其推断出的函数类型也会<!--
-->没有接收者（它会有一个接受接收者对象的额外参数）。如需改为<!--
-->带有接收者的函数类型，请明确指定其类型：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
val isEmptyStringList: List<String>.() -> Boolean = List<String>::isEmpty 
```
</div>

### 示例：函数组合

考虑以下函数：

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
fun <A, B, C> compose(f: (B) -> C, g: (A) -> B): (A) -> C {
    return { x -> f(g(x)) }
}
```
</div>

它返回一个传给它的两个函数的组合：`compose(f, g) = f(g(*))`。
现在，你可以将其应用于可调用引用：

<div class="sample" markdown="1" theme="idea">

```kotlin
fun <A, B, C> compose(f: (B) -> C, g: (A) -> B): (A) -> C {
    return { x -> f(g(x)) }
}

fun isOdd(x: Int) = x % 2 != 0

fun main() {
//sampleStart
    fun length(s: String) = s.length
    
    val oddLength = compose(::isOdd, ::length)
    val strings = listOf("a", "ab", "abc")
    
    println(strings.filter(oddLength))
//sampleEnd
}
```
</div>

### 属性引用

要把属性作为 Kotlin中 的一等对象来访问，我们也可以使用 `::` 运算符：

<div class="sample" markdown="1" theme="idea">

```kotlin
val x = 1

fun main() {
    println(::x.get())
    println(::x.name) 
}
```
</div>

表达式 `::x` 求值为 `KProperty<Int>` 类型的属性对象，它允许我们使用
`get()` 读取它的值，或者使用 `name` 属性来获取属性名。更多信息请参见<!--
-->[关于 `KProperty` 类的文档](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-property/index.html)。

对于可变属性，例如 `var y = 1`，`::y` 返回 [`KMutableProperty<Int>`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-mutable-property/index.html) 类型的一个值，
该类型有一个 `set()` 方法。

<div class="sample" markdown="1" theme="idea">

```kotlin
var y = 1

fun main() {
    ::y.set(2)
    println(y)
}
```       
</div>            

属性引用可以用在预期具有单个泛型参数的函数的地方：

<div class="sample" markdown="1" theme="idea">

```kotlin
fun main() {
//sampleStart
    val strs = listOf("a", "bc", "def")
    println(strs.map(String::length))
//sampleEnd
}
```
</div>

要访问属于类的成员的属性，我们这样限定它：

<div class="sample" markdown="1" theme="idea">
```kotlin
fun main() {
//sampleStart
    class A(val p: Int)
    val prop = A::p
    println(prop.get(A(1)))
//sampleEnd
}
```
</div>

对于扩展属性：

<div class="sample" markdown="1" theme="idea" auto-indent="false">

```kotlin
val String.lastChar: Char
    get() = this[length - 1]

fun main() {
    println(String::lastChar.get("abc"))
}
```
</div>

### 与 Java 反射的互操作性

在 JVM 平台上，标准库包含反射类的扩展，它提供了与 Java
反射对象之间映射（参见 `kotlin.reflect.jvm` 包）。
例如，要查找一个用作 Kotlin 属性 getter 的 幕后字段或 Java方法，可以这样写：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlin.reflect.jvm.*

class A(val p: Int)

fun main() {
    println(A::p.javaGetter) // 输出 "public final int A.getP()"
    println(A::p.javaField)  // 输出 "private final int A.p"
}
```
</div>

要获得对应于 Java 类的 Kotlin 类，请使用 `.kotlin` 扩展属性：

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
fun getKClass(o: Any): KClass<Any> = o.javaClass.kotlin
```
</div>

### 构造函数引用

构造函数可以像方法和属性那样引用。他们可以用于期待这样的函数类型对象的任何<!--
-->地方：它与该构造函数接受相同参数并且返回相应类型的对象。
通过使用 `::` 操作符并添加类名来引用构造函数。考虑下面的函数，
它期待一个无参并返回 `Foo` 类型的函数参数：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
class Foo

fun function(factory: () -> Foo) {
    val x: Foo = factory()
}
```
</div>

使用 `::Foo`，类 Foo 的零参数构造函数，我们可以这样简单地调用它：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
function(::Foo)
```
</div>

构造函数的可调用引用的类型也是
[`KFunction<out R>`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-function/index.html) 的子类型之一
，取决于其参数个数。

{:#绑定的函数与属性引用自-11-起}
## 绑定的函数与属性引用（自 1.1 起）

你可以引用特定对象的实例方法：

<div class="sample" markdown="1" theme="idea">

```kotlin
fun main() {
//sampleStart
    val numberRegex = "\\d+".toRegex()
    println(numberRegex.matches("29"))
     
    val isNumber = numberRegex::matches
    println(isNumber("29"))
//sampleEnd
}
```
</div>

取代直接调用方法 `matches` 的是我们存储其引用。
这样的引用会绑定到其接收者上。
它可以直接调用（如上例所示）或者用于任何期待一个函数类型表达式的时候：

<div class="sample" markdown="1" theme="idea">

```kotlin
fun main() {
//sampleStart
    val numberRegex = "\\d+".toRegex()
    val strings = listOf("abc", "124", "a70")
    println(strings.filter(numberRegex::matches))
//sampleEnd
}
```
</div>

比较绑定的类型和相应的未绑定类型的引用。
绑定的可调用引用有其接收者“附加”到其上，因此接收者的类型不再是参数：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
val isNumber: (CharSequence) -> Boolean = numberRegex::matches

val matches: (Regex, CharSequence) -> Boolean = Regex::matches
```
</div>

属性引用也可以绑定：

<div class="sample" markdown="1" theme="idea">
```kotlin
fun main() {
//sampleStart
    val prop = "abc"::length
    println(prop.get())
//sampleEnd
}
```
</div>

自 Kotlin 1.2 起，无需显式指定 `this` 作为接收者：`this::foo` 与 `::foo` 是等价的。

### 绑定的构造函数引用

[*inner*{: .keyword} 类](nested-classes.html#内部类)的构造函数的绑定的可调用引用可<!--
-->通过提供外部类的实例来获得：

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
class Outer {
    inner class Inner
}

val o = Outer()
val boundInnerCtor = o::Inner
```
</div>

