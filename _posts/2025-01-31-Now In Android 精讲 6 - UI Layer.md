---
highlight: androidstudio
theme: fancy
excerpt: "深入解析Android应用的UI层架构设计！探讨如何使用Jetpack Compose构建高性能、可维护的用户界面，并揭示现代Android开发的最佳实践。"
---

## 界面层（UI Layer）概览

界面层的主要作用是展示数据，并且响应用户交互。下面的界面层官方架构图告诉我们，界面层是属于整个应用的最上层，他主要由 UI element 和 state holder 组成。 UI 元素通过从 State holders 获取 State 从而向用户提供可交互的 UI，那么什么是 State？

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/6480f513f7bc456da9da8e87fe5c17de~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1741348332&x-orig-sign=ezlCMqUOA9wse2ip8gyYD9Hew2g%3D)

## UI state 定义

> if the UI is what the user sees, the UI state is what the app says they should see. Like two sides of the same coin, the UI is the visual representation of the UI state. Any changes to the UI state are immediately reflected in the UI.

官方对 UI state 介绍非常抽象，换成通俗意义上的话来说，UI 是给用户看的，UI State 是给 app 看的。UI state 首先告诉 app，app 才能告诉你该长啥样。

那么通常一个 UI state 应该长啥样？举个例子：

```kotlin
val topicUiState: StateFlow<TopicUiState> = topicUiState(
    topicId = topicId,
    userDataRepository = userDataRepository,
    topicsRepository = topicsRepository,
)
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = TopicUiState.Loading,
    )
```

一般来说 UI state 首要满足的就是不可变对象，多数情况我们使用 val 对外提供一个`不可变`的对象，其次他的命名习惯的是：功能 + UiState。例如在本例中是一个 topic 相关的状态，那么项目里面起名就叫 TopicUiState。

## 如何管理 UI state？

![图二](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/8142171e81c746548536558ca99e819c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1741348332&x-orig-sign=RlVs0Oe40P8TU5lxtrU2JcjjzDM%3D)

整个应用如上图所示，都应该遵循 Unidirectional Data Flow (UDF) 来管理 UI state, 按照图中所示即是：状态向下传递，事件向上传递。我们换种方式来理解即是，UI state 负责驱动 UI 的变化，但是 UI 的修改需要通过 event 传递给上层处理。state 跟 event 他们一个负责通知 UI 的变化，一个负责传递用户意图。单项数据流很好的把他们的职责分离出来。

**NOTE：我们在谈论单向数据流的时候不要脱离了 UI layer，如果你在其他层，例如 Data layer 或是 Domain layer，我想是一件不合适的事情，因为他们本身就需要从本地获取/发送数据，进行各种复杂的交互。**

## 通过 Now In Android 里面的代码来学习如何向外提供数据流

```kotin
val uiState: StateFlow<InterestsUiState> = combine(
    selectedTopicId,
    getFollowableTopics(sortBy = TopicSortField.NAME),
    InterestsUiState::Interests,
).stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5_000),
    initialValue = InterestsUiState.Loading,
)


sealed interface InterestsUiState {
    data object Loading : InterestsUiState

    data class Interests(
        val selectedTopicId: String?,
        val topics: List<FollowableTopic>,
    ) : InterestsUiState

    data object Empty : InterestsUiState
}
```

一般来说我们通过向外暴露一个 `StateFlow` 类型的可观察数据流，对外提供不可变的数据。在应该也会有一些小伙伴在平时写的时候会贪图省事，使用 MutableStateFlow ，这样既可以操作，又可以提供数据，这样其实是破坏封装的，向使用方提供了过多的能力，会使得代码维护变得困难。

### 如何合理的提供 state？

1.  首先 state 内部的各个 property 应该有相关性，举个例子：上面的代码 InterestsUiState 里面 Interests 提供的都是 Interest 相关内容，如果里面提供了订阅的信息，那应该甚为不妥。其次如果 state 某个 property 更新频率很高，那么 state 更新频率会变的很高，会引起很多 compose 不必要的重组，降低 UI 性能。这时候可以选择将这个 property 单独提取出去，再行暴露给调用方。
2.  尝试使用 [`distinctUntilChanged()`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/distinct-until-changed.html)，或者其他操作符减少不必要的更新

## 如何理解 Event？

我们先看一段官方的介绍

> *UI events* are actions that should be handled in the UI layer, either by the UI or by the ViewModel.

`Event/事件` 是 UI 层应该处理的操作，注意这里面说的 UI 层而不只是 UI element，他可能是 UI 亦或是 ViewModel。

### Event 有哪些类型？

*   viewmodel 事件：ViewModel 事件应始终会引发界面状态更新。这里面官方文档的[中文翻译](https://developer.android.com/topic/architecture/ui-layer/events?hl=zh-cn#consuming-trigger-updates) 有一处不够准确，
    英文原意是：Consuming events can trigger state updates，翻译成中文就变成了：使用事件可能会触发状态更新。can 这个词翻译成`可`或者是`能`都行，唯独`可能`不行，这地方语义应该翻译成`会`。
*   导航事件：调用导航控制器路由到指定 composable screen。\
    参考下面项目里面的代码 onTopicClick = navController::navigateToTopic

    ```kotlin
    fun NiaNavHost(
        appState: NiaAppState,
        onShowSnackbar: suspend (String, String?) -> Boolean,
        modifier: Modifier = Modifier,
    ) {
        val navController = appState.navController
        NavHost(
            navController = navController,
            startDestination = ForYouBaseRoute,
            modifier = modifier,
        ) {
            forYouSection(
                onTopicClick = navController::navigateToTopic,
            ) 
            ...
            interestsListDetailScreen()
        }
    }
    ```

## 状态容器与状态管理

### 逻辑

从前面一节里面我们学习到，状态是由事件驱动的，一般由 viewmodel 进行更新，用通俗的话来讲事件调用的负责更新或者生成新的状态的流程我们称之为 `逻辑`。\
上面一节讲到了事件有 viewmodel 事件和导航事件，一般我们把 viewmodel 事件触发的逻辑称之为业务逻辑，导航事件触发的逻辑称之为界面逻辑。业务逻辑通常不依赖具体的生命周期（你是不是从来没在 viewmodel 里面依赖过 lifecyle 之类的代码），界面逻辑依赖生命周期（页面没了逻辑自然也不能执行）。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/625076cfed0742e0af2f4b09acc3ec81~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1741348332&x-orig-sign=ybcfwfmEa2N1txcpaub4vXIlQoY%3D)

### 状态容器

一般来说，如果不是非常简单的界面，我们都会有一个状态容器来存储状态。然后与逻辑类似，不依赖界面生命周期的称之为业务状态逻辑容器，依赖界面的我们称之为界面状态逻辑容器。

### 业务逻辑状态容器 ViewModel

平时我们接触到最多的 viewmodel 就是`业务状态逻辑容器`，这个应该很好理解，viewmodel 本身不依赖界面，他依赖上游的 Data layer 或是 Domain layer 的能力，保存状态，作为业务逻辑的中转站而存在。（在依赖关系里面是 viewmodel 处于 UI 的上游，一般由 fragment 或者 compose 依赖 viewmodel，所以他的生命周期往往要长于 UI）
看个例子：

```kotlin
class InterestsViewModel @Inject constructor(
   private val savedStateHandle: SavedStateHandle,
   val userDataRepository: UserDataRepository,
   getFollowableTopics: GetFollowableTopicsUseCase,
) : ViewModel() {

   // Key used to save and retrieve the currently selected topic id from saved state.
   private val selectedTopicIdKey = "selectedTopicIdKey"

   private val interestsRoute: InterestsRoute = savedStateHandle.toRoute()
   private val selectedTopicId = savedStateHandle.getStateFlow(
       key = selectedTopicIdKey,
       initialValue = interestsRoute.initialTopicId,
   )

   val uiState: StateFlow<InterestsUiState> = combine(
       selectedTopicId,
       getFollowableTopics(sortBy = TopicSortField.NAME),
       InterestsUiState::Interests,
   ).stateIn(
       scope = viewModelScope,
       started = SharingStarted.WhileSubscribed(5_000),
       initialValue = InterestsUiState.Loading,
   )
   

   fun followTopic(followedTopicId: String, followed: Boolean) {
       viewModelScope.launch {
           userDataRepository.setTopicIdFollowed(followedTopicId, followed)
       }
   }

   fun onTopicClick(topicId: String?) {
       savedStateHandle[selectedTopicIdKey] = topicId
   }

```

从上面的例子我们可以看到一个典型的 viewmodel 可能要处理页面重建，负责与 Data layer 以及 domain 交互，UI 提供 state 的职责，是一个非常重要的角色。

#### 如何在 UI 中使用 state 和调用业务逻辑？

我们先看一段项目里面的代码：

```kotlin
@Composable
fun InterestsRoute(
    onTopicClick: (String) -> Unit,
    modifier: Modifier = Modifier,
    highlightSelectedTopic: Boolean = false,
    viewModel: InterestsViewModel = hiltViewModel(),
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    InterestsScreen(
        uiState = uiState,
        followTopic = viewModel::followTopic,
        onTopicClick = {
            viewModel.onTopicClick(it)
            onTopicClick(it)
        },
        highlightSelectedTopic = highlightSelectedTopic,
        modifier = modifier,
    )
}

@Composable
internal fun InterestsScreen(
    uiState: InterestsUiState,
    followTopic: (String, Boolean) -> Unit,
    onTopicClick: (String) -> Unit,
    modifier: Modifier = Modifier,
    highlightSelectedTopic: Boolean = false,
) {
   ...
}
```

从上面的代码我们可以看出来，状态的获取是通过 val uiState by viewModel.uiState.collectAsStateWithLifecycle() 这样的方式获取，然后与生命周期绑定。然后只向 compose 传递需要的参数和 Event，而不是整个 `viewmodel`。这一点非常重要，如果你的组件依赖了 viewmodel 那么你的组件的可复用性会变得很低。

#### 严禁向 compose 方法传递 `viewmodel`

在这里我们稍微看一段上面的 viewmodel 的代码：

```kotlin
class InterestsViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    val userDataRepository: UserDataRepository,
    getFollowableTopics: GetFollowableTopicsUseCase,
) : ViewModel()
```

可以看到他依赖了 Data layer 和 Domain layer 里面的部分 class，如果我一个 compose 的组件依赖了 viewmodel 那么，他是不是也需要依赖这些文件，这无疑让其使用这个组件的人增加了成本，而且这破坏了单一职责。再看看上面的 `InterestsScreen` ，这种方式是不是比之直接传递 viewmodel 要好上许多，复用性也更强了。

### 界面逻辑及其状态容器

界面逻辑我们平时开发中接触也不少，例如导航，获取图片资源。这些操作依赖于界面的存在，如果界面不存在了导航，获取图片资源这些便也没了意义。那么保存界面逻辑状态的容器便不需要很复杂，可以使用普通类来保存状态。

```kotlin
@Composable
fun rememberNiaAppState(
    networkMonitor: NetworkMonitor,
    userNewsResourceRepository: UserNewsResourceRepository,
    timeZoneMonitor: TimeZoneMonitor,
    coroutineScope: CoroutineScope = rememberCoroutineScope(),
    navController: NavHostController = rememberNavController(),
): NiaAppState
```

这是一段项目里面的的界面状态容器代码，可以看到在 compose 里面界面容器本身也是 compose 的一部分，是可组合方法，那么其必然是可以被其他可组合方法复用的。

### 如何选择状态容器的类型？

一般来说业务逻辑我们会选择 viewmodel，跟 app 页面依赖较多的例如资源文件、导航控制器等我们可以选择使用普通类。

## 状态（state）、状态容器（state holder）、事件（event）如何串起来？

先来看一段状态生产的示意图：
![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/a078478331d945c4ac2947f9e0d28cae~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1741348332&x-orig-sign=hHb3aDDkfC4a0YpVMxfZwAJ%2FwHk%3D)\
首先`事件`他的来源可能是来自 viewmodel 依赖的 domain、data layer 的数据变化，也有可能是用户的交互操作，这些统一作为输入，然后 state holder，处理更新状态。这些事件在整个过程中我们称之为 `输入`，而输入会交给 viewmodel 这样的 state holder 来处理，然后产生 UI state。

### 输入输出特点

举个例子：

```kotlin
val feedUiState: StateFlow<NewsFeedUiState> =
    userNewsResourceRepository.observeAllBookmarked()
        .map<List<UserNewsResource>, NewsFeedUiState>(NewsFeedUiState::Success)
        .onStart { emit(Loading) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = Loading,
        )
```

首先 viewmodel 也就是状态容器接受其他层的数据变化，整个过程是在异步的不在主线程执行，然后返回一个可观察的 stateflow。当然在实际开发过程中接受的输入不仅仅是 flow，其他类型的操作，或者两者混合使用都是可以的，灵活运用。

### 状态生产过程的初始化？

本着最小化调用的原则，生产过程应该在需要的时候才启动。所以我们应该尽可能延迟状态生成流水线的初始化，以节省系统资源。\
在这里面有一个细节就是**避免在 viewmodel 的构造或者 init 方法里面调用异步方法**。这个应该是很多人都忽略的一点，首先如果调用异步方法，那么这个方法的生命周期可能会长于 viewmodel 自身，导致对象的泄露。其次根据函数式编程的理念就是不产生副作用，在构造或者 init 里面调用异步代码显然是不符合这个规范的。如果使用异步编程，不能保证构造函数的幂等性。而且有可能对象还没创建完就调用部分方法，产生不可预期的问题。\
在学习 Now In Android 项目的时候我看到，每个 viewmodel 都是这么设计的，希望我们以后都能避免犯类似的错误。\
下面举例说：

```kotlin
// 反面示例：不推荐的异步初始化
class BadViewModel(
    private val userRepository: UserRepository
) : ViewModel() {
    // 错误：在构造函数中直接启动异步操作
    init {
        viewModelScope.launch {
            // 可能导致状态不一致和生命周期问题
            val userData = userRepository.fetchUserData()
            _userState.value = userData 
        }
    }
}

// 正面示例：推荐的异步操作方式
class GoodViewModel(
    private val userRepository: UserRepository
) : ViewModel() {
    private val _userState = MutableStateFlow<UserData?>(null)
    val userState: StateFlow<UserData?> = _userState.asStateFlow()

    // 提供显式的加载方法
    fun loadUserData() {
        viewModelScope.launch {
            try {
                val userData = userRepository.fetchUserData()
                _userState.value = userData
            } catch (e: Exception) {
                // 处理错误
                _userState.value = null
            }
        }
    }
}
```

## The end，本章完

在这里给您拜个晚年，祝您晚年生活愉快 😄
