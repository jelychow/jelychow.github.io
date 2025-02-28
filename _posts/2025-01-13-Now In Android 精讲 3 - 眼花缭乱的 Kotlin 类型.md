---
theme: fancy
highlight: a11y-light
---
## 前言
在开始阅读之前有一个问题想问问大家是否都了然于胸。请问 interface，fun interface，sealed interface，sealed class，enum，object，data object他们之间有什么联系，每个类型有什么特定的使用场景呢？  
如果上面的问题你都如数家珍，那么你可以跳过本章节。

## 正篇
我相信很多第一次接触 Now In Android 这个项目的同学都或多或少对里面各种 kotlin 类型迷的眼花缭乱。很多时候会有这种感觉，原来可以这么用啊，可是有时候想想为什么却不是很清楚。下面我会分享一些我在学习 Now In Android 过程中对一些常见的数据类型的思考。

### 第一问：interface 与 fun interface 有什么关系？到底该用哪个？
相信在日常的 coding 过程中大家或多或少都会遇见这种情况，编译器会提示你可以使用 fun interface 代替 interface。  
关于 fun interface 的介绍可以看这个[链接](https://kotlinlang.org/docs/fun-interfaces.html)，借用官方文档的话来说一下重点：
1. **只有一个抽象成员函数的接口称为功能接口，或单一抽象方法（SAM/ Single abstract method）接口。功能接口可以有多个非抽象成员函数，但只能有一个抽象成员函数**
2. **对于功能接口，您可以使用 SAM 转换，通过使用lambda 表达式，可以使您的代码更简洁、更易读。**  
   如何理解上面的话呢，首先满足这个 SAM 转换需要 2 个条件，只有一个抽象方法，还需要使用 fun interface 开头。那么他优点到底是怎么样的呢，下面我用代码来展示说明：

```kt
fun interface Printer {
    fun print(text: String)
}

val huaweiPrinter = object :Printer{
    override fun print(text: String) {
        println("Huawei Printer is printing $text")
    }
}

val xiaomiPrinter = Printer{
    println("Xiaomi Printer is printing $it")
}
```
可以很直观的看到使用 SAM 转换带来的优势，写法非常简洁。看到这里你会不会发出感叹：就这？  
实际上 fun interface 本身可以看成是一种函数类型，可以配合高阶函数一起使用


```kt
fun usePrinter(text: String,printer: Printer) {
    printer.print(text)
}

fun main() {
    // 常规写法
    usePrinter("r u ok",xiaomiPrinter)
    // 方法引用
    usePrinter("yao yao ling xian",huaweiPrinter::print)
    // 高阶函数+方法引用
    usePrinter("yao yao ling xian",Printer(huaweiPrinter::print))
    usePrinter("excellent",Printer{
        print("normal printer is printing $it")
    })
    // lambda 优化
    usePrinter("not bad"){
        print("this printer is $it")
    }
}
```
上面的代码为 kotlin 的接口使用带来新的变化，可以看出来接口的使用更灵活了。但是其原因为什么呢？个人看来简化代码不是最重要的，对**函数是编程的增强**才是变化的最主要目的。
本节还有最后一个问题，interface 和 fun interface 到底用哪个？  
如果你的 interface 满足 SAM，那么你就按照编辑器给你的提示加上一个 fun，这样会有更多的灵活性，如果你的 interface 有多个抽象方法，那么只能使用 interface。

#### fun interface 在 Now In Android 里的实践
讲了这么多我们看下 Now In Android 里面的 fun interface 用法，看看你能不能看得懂。不得不说现在的 kotlin 早就不是一门简单入门的语言了，越来越抽象了，Android 程序员没有温柔 🥲。


```kt
    fun interface DemoAssetManager {
        fun open(fileName: String): InputStream
    }
    
    @Provides
    @Singleton
    fun providesDemoAssetManager(
        @ApplicationContext context: Context,
    ): DemoAssetManager = DemoAssetManager(context.assets::open)
```
### 第二问：interface ，sealed interface，sealed class 有什么关系？到底该用哪个？
在学习新版的 Now In Android 项目的时候现在几乎（保守的说）看不到 sealed class 在使用了，取而代之的是 sealed interface。那么 sealed interface 到底有什么魅力，让他受到如此推崇。
#### **case 1**：
现在假设你是一个库的开发者，你希望你的 interface 不仅仅是在库内部被使用，而不希望被外部集成滥用。因为库的变化，某种情况是黑盒子，你只需要依赖他就行了，不需要对其中实现了解太多。但是如果你使用 interface 设计，那么不可避免的会被使用方一顿操作，这对会库设计者是不可接受。这时候你会不会想既可以保留接口的组合设计，又可以保证他们仅仅在库的范围内实现？
#### **case 2**：
假设你有一个 class 现已继承 parent class ，但是亦想限制其子类实现在模块范围之内，如果是你会打算怎么设计呢？

这时候 sealed interface 就应运而生了，可以说 kotlin 作为一门开源的语言是充分听取了开发者的呼声。当然这也是一把双刃剑，功能强大的时候，上手的门槛也在逐步提高。那么在 Now In Android 这个项目里面 sealed interface 主要有什么用法呢？下面我使用代码展示一下：


```kt
sealed interface Result<out T> {
    data class Success<T>(val data: T) : Result<T>
    data class Error(val exception: Throwable) : Result<Nothing>
    data object Loading : Result<Nothing>
}
sealed interface TopicUiState {
    data class Success(val followableTopic: FollowableTopic) : TopicUiState
    data object Error : TopicUiState
    data object Loading : TopicUiState
}

sealed interface NewsUiState {
    data class Success(val news: List<UserNewsResource>) : NewsUiState
    data object Error : NewsUiState
    data object Loading : NewsUiState
}
```
可以看到 sealed interface 主要用来表达一些有限状态的场景，在很多时候有类似 Enum 的作用，他的写法也更简单了。当然好奇的宝宝肯定要问啦，sealed interface 跟 enum 到底有什么区别呢？请别急，我们移步下一节。
### 第三问 sealed interface，enum 到底怎么选？
#### Now In Android enum 的使用方式：

```kt
enum class FlavorDimension {
    contentType
}

enum class NiaFlavor(val dimension: FlavorDimension, val applicationIdSuffix: String? = null) {
    demo(FlavorDimension.contentType, applicationIdSuffix = ".demo"),
    prod(FlavorDimension.contentType),
}

enum class DarkThemeConfig {
    FOLLOW_SYSTEM,
    LIGHT,
    DARK,
}
```
从上面的代码我们可以看出来，enum 只能接受常量作为参数，一般来说要大写（当然不是强制的，代码习惯问题）
#### Now In Android 中 sealed interface 的使用方式：


```kt
/**
 * A sealed hierarchy describing the state of the feed of news resources.
 */
sealed interface NewsFeedUiState {
    /**
     * The feed is still loading.
     */
    data object Loading : NewsFeedUiState

    /**
     * The feed is loaded with the given list of news resources.
     */
    data class Success(
        /**
         * The list of news resources contained in this feed.
         */
        val feed: List<UserNewsResource>,
    ) : NewsFeedUiState
}

```
可以看到 sealed interface 表示页面 state，在 loading 到 success 之间反复横跳毫无压力，他的典型的使用场景就是作为一个页面的 state 来使用，也可以作为网络请求结果来使用。涉及到不同状态之间切换，并且有数据交互的，多选用 sealed interface。  
到这里你或许有点疑问，既然二者用法如此之象，为何不合二为一，用 sealed interface 来代替 enum class，这个想法说实话已经有已经有好奇宝宝向 kotlin 的提了这个 [issue](https://youtrack.jetbrains.com/issue/KT-47868), 至于未来会不会出现更简单的 sealed interface 写法，让我们拭目以待。
#### enum 与 sealed interface 的最佳实践
sealed interface 在很多时候可以完成 enum 的功能，但是反之是不可以的。
- 一般来说 enum 代表一组固定的值（所以需要大写）
- sealed interface 常用来表示一组特定（例如 sucess，fail）的值
>总的来说 sealed interface 很强大，不过现阶段他们两个不是完全等价的，写的时候我们需要按照每个数据类型的作用来选择。

### 第四问 object，data object 傻傻分不清楚？
在阅读本节之前，如果还没了解过 data object 一定要看一下[官方文档](https://kotlinlang.org/docs/object-declarations.html#data-objects)，关于 data object 的 api 以及使用说明，我这里就不展开介绍了。我们把重点放在他的设计目标以及如何正确的使用上。其实关于这个 data object 也有不少人向 kotlin 官方[提问](https://github.com/Kotlin/KEEP/issues/317)，不过目前看起来 data object 主要设计目的主要有 2 点：
1. 为了在 sealed class/interface 里面声明一致性（更好的可读性）

    ```kt
    sealed interface NewsUiState {
        data class Success(val news: List<UserNewsResource>) : NewsUiState
        object Error : NewsUiState
        object Loading : NewsUiState
    }

    sealed interface NewsUiState {
        data class Success(val news: List<UserNewsResource>) : NewsUiState
        data object Error : NewsUiState
        data object Loading : NewsUiState
    }
    ```
   对比一下，是不是看起来好多了
2. 简化语法，为将来实现类 enum 的语法作打算
#### data object，object 到底怎么用？
我们参考 Now In Android 里面的示例，
- data object 一般配合 sealed interface/class 作无状态的全局单例来使用
- object 一般用来定义常量，或者工具类之类来使用

```object示例代码
object NiaTagDefaults {
    const val UNFOLLOWED_TOPIC_TAG_CONTAINER_ALPHA = 0.5f
    const val DISABLED_TOPIC_TAG_CONTAINER_ALPHA = 0.12f
}
```
## 结尾
谢谢观看😄