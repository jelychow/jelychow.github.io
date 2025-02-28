---
theme: smartblue
excerpt: "深入解析Android应用架构的核心技术！详细讲解模块化设计和依赖注入，帮你构建更加解耦、可测试和可维护的Android应用。"
---
在讲解之前，先推荐大家**一定一定一定**要看一下官方对于架构的总结，然后回来阅读本文，当然也可以不回来 🥲。
1. [Android 官方: **模块化简介**](https://developer.android.com/topic/modularization?hl=zh-cn)
2. [Now In Android: **Modularization learning journey**](https://github.com/android/nowinandroid/blob/main/docs/ModularizationLearningJourney.md)

## 模块化开发的目的

组件化的首要目的是增加项目的可维护性。当然可维护性只是一个范围比较大词儿，听起来些许的空洞，他可能涉及到很多方方面面，但是确实用来形容模块化目的最好的词。在无数次的工程实践中我们会逐渐发现问题，也会尝试解决问题，对于 Android 项目的不断膨胀的 feature，不断迭代的版本，我们也慢慢发现模块化是目前最好的解药。

## 模块化开发的好处
- **可重用性**  
  对于某些特定 features 实际开发过程可能会提供给数个 app 来编译，这会在很多程度上节省人力，增加组织间的一致性
- **严格控制可见性**：  
  这一点其实是被大多数 Android 开发忽视的一点。试想一下如果你的项目结构虽然也进行了模块化配置，但是你的 module 里面的每个 class 都对外暴露，这样会让其他使用方感觉到奇怪，会产生很多预期之外的行为。下面举例说明说明一下：

```kt
sealed interface LocationState {
    data object Accurate : LocationState
    data object Coarse : LocationState
    data object None : LocationState
}

class LocationProvider {
    
    fun getLocation(state: LocationState) {
        when (state) {
            LocationState.Accurate -> getAccurateLocation()
            LocationState.Coarse -> getCoarseLocation()
            LocationState.None -> throw IllegalStateException("No permission")
        }
    }
    
    fun getAccurateLocation() {}

    fun getCoarseLocation() {}
}
```
假设我们有一个 module 向外提供地址服务，如果有精确位置我们就提供精确位置，没有我们就提供粗略位置。想想就知道向外暴露出这样一个服务是非常危险的，调用方通常不会像我们想的那样调用 api，他可能直接就调用 getAccurateLocation() 这时候如果没有 permission，那么你肯定会打一个🤧。我想聪明的你肯定会在 fun getAccurateLocation() 和 fun getCoarseLocation() 加上一个 `private`。  
可能很多人渐渐忘了 open, internal, private 这些修饰符的作用了。下面我们一起简单的回忆一下：
### open 关键字
默认情况下，Kotlin 中的类和方法都是 `final`，这意味着它们不能被继承或重写。使用 `open` 关键字可以允许类或方法被继承或重写。  
在日常开发的时候我们多少都会遇到这样一个场景：我们有两个 viewmodel 逻辑非常相似，需要复用里面的部分逻辑。这个时候你可能会使用一个 `open` 关键字来提取一个 BaseViewModel，然后你继承 使用 ViewModelImp1，ViewModelImp2。可能你觉得这是一个复用的好方案，但是我建议你在开发过程非必要切勿使用 `open` 。只要稍微想一下就会明白，你暴露的 baseviewmodel 很容易被调用方意外使用，出现一些预期之外的错误。总结下`open` 有如下几个缺陷
1. 类的继承层次复杂  
   如果你将类标记为 `open`，其他开发者可以自由地继承这个类并重写方法。这可能会导致复杂的继承层次，使得代码难以理解和维护。我们平时说 多用组合少用继承也是这个原因。
2. 不可预见的行为  
   当类的方法被重写时，可能会引入不可预见的行为。这使得调试变得更加困难，特别是在大型团队中，其他开发者可能不清楚重写的方法会如何影响整体逻辑。
3. 破坏封装  
   使用 `open` 关键字可能导致类的实现细节暴露给外部，打破了封装原则。这使得类的内部实现不再独立，增加了修改时引入错误的风险。  
   说了这么多 open 关键字的坏话，怪不好意思的🫣，那么有没有适合他的场景呢，sorry 我暂时没想到，如果你有的话，欢迎留言告诉我😄。
#### Now In Android 的实践
首先我们来看一段(VCR/ 扣德：

```kt
@HiltViewModel
class ForYouViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    syncManager: SyncManager,
    private val analyticsHelper: AnalyticsHelper,
    private val userDataRepository: UserDataRepository,
    userNewsResourceRepository: UserNewsResourceRepository,
    getFollowableTopics: GetFollowableTopicsUseCase,
) : ViewModel() {
```
代码里面通过依赖注入来进行 IOC，专业的人做专业的事，不同模块处理不同的能力，通过组合来代替继承，顺带提一下，在最新 Android 推荐的官方的架构分层里面，`UI 层，Domain 层，Data 层`。一般来说 Domain 层不是必须的，但是如果有多个 ViewModel 重复使用的简单业务逻辑，官方建议的都这些功能都放入一个 UseCase 之中。这是一个约定俗成的规范，当你在阅读别人代码的时候，当你看到一个 xxxUseCase 的时候，你马上就明白，这是个 xxx 功能的集合。
他的推荐命名方式是：动词 + 名词/目标 (选填) + UseCase。
### internal 关键字
简单的来说 internal 就是限制修饰的方法/class 可见性仅限于当前模块内，相当于模块内的 public，模块外的 private。通常来说模块内向其他 class、方法提供能力需要添加 internal 修饰符，这样才能最小化暴露使用能力。  
在 Now In Android 里面大量使用了 internal 来修饰 class，很大程度保障了，模块的稳定性。

### private 关键字
`private` 限制当前修饰的变量作用域仅限于当前文件/class。通常来说只要这要该方法不向外提供服务都应该用`private`，这样能够尽量减少预期之外的行为。
## 模块化的划分原则
> 注意 下面这段文字摘抄自官方文档，写的太好了，建议收藏，做心法背诵体会，搞不好可以结丹。

表示模块化代码库的一种方式是使用**耦合**和**内聚**属性。耦合用于衡量模块相互依赖的程度。在本指南中，内聚用于衡量单个模块的不同元素在功能上的相关性。一般而言，应尽力实现低耦合和高内聚：

-   **低耦合**是指模块应尽可能相互独立，这样一来，对一个模块所做的更改将对其他模块产生零影响或极小的影响。**模块相互之间不应了解对方的内部运行原理**。
-   **高内聚**是指多个模块的代码集合应当像一个系统一样运行。它们应具有明确定义的职责，并始终位于特定领域知识范围以内。假设有一个电子书应用示例。在同一个模块中融合图书相关代码和付款相关代码是不合适的，因为图书和付款是两个不同的功能领域。

**提示**：如果两个模块相互严重依赖对方的信息，则可能表明这两个模块应当作为一个系统运行。相反，如果一个模块的两个部分并不经常相互交互，则这两个部分应当成为两个独立模块。  
**举例来说明**：例如 feature:bookmarks 这个模块，内部所有的文件理论上都只跟 bookmark 相关，如果有无关的内容需要移除到相应模块，他们在内部实现**高内聚**，在外部他与其他模块之间尽量减少依赖

```kt
dependencies {
    implementation(projects.core.data)
    testImplementation(projects.core.testing)
    androidTestImplementation(libs.bundles.androidx.compose.ui.test)
    androidTestImplementation(projects.core.testing)
}
```
这个就叫做**低耦合**
## 模块通信与依赖注入
通常来说模块之间应该避直接通信，例如 moduleA 直接 moduleB，这样容易形成循环依赖。现在通常的是做法是下沉一个公共模块，暴露接口层，给需要使用的 module。Android 的官方文档里面，提到了 2 种解耦的方式：
1. DI 依赖注入  
   依赖注入是官方大力推荐的方式，通过注入 接口 来实现 IOC。依赖注入的框架有很多，主流的有 Dagger，Hilt，Koin，在这里官方给出了好多使用他的理由，但是对于我个人的理解来说 DI 最方便的就是单元测试比较容易，如果你平时也写单元测试的话，你会很有体会，mock 一个对象 run 一个 test 相当 easy。第二个优势就是DI 框架会处理生命周期，scope 相关的逻辑，能够帮我自动规避一些问题。不得不说 DI 这种解耦方式已经成为当今 Android 发展的主流，如果我们浏览一些国外的技术网站，开一些开源代码我们会发现，DI 出镜其实非常之高。  
   DI 的功能越强，概念自然越多，学习的门槛有点高，在这里我建议，大家学习由点及面，启动一个小项目，用到啥的时候学习啥，配合一下 gpt 慢慢的就可以掌握了，掌握之后内心 OS：也挺简单的🤣。
2. Service Locator 服务发现  
   这是当前国内 Android 最为熟知的一种模式，以 Arouter 为代表的 Service Locator 曾经是我的最爱，这种开发模式的最大特点就是上手异常的简单。只需要简单的定义，然后 ServiceLocator 找一下就 OK。不过这种方式单元测试没有 DI 那么容易，对于 scope 和生命周期处理也没有 Hilt 之类的库强大。
### 开发者到底该使用哪个？
对于大部分 Android 开发者来说，如果不想落后于时代，以后还想有口饭吃，自然是要跟着官方走的，没必要守着以前的那套框架不肯退出来。在这里我呼吁大家趁早切换到 DI 框架上来，其实也没那么麻烦。当然关于 Hilt 或者 Koin 等框架使用可以看相关文档，那里有更细致的讲解。

## 模块化最佳实践
实际开发过程中，第一个应该避免的就是过度模块化，这某种程度来说危害更大，给实际维护工作造成很多不必要的负担。模块化的粒度要适中，给未来演进留下空间。第二避免模块化太复杂，避免使用各种奇技淫巧在模块化上，省的你在以后打很多喷嚏。
## 本章完，我们下次见😊