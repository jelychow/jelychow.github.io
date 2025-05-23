---
highlight: vs2015
theme: smartblue
excerpt: "Navigation 3 介绍"
---
# Navigation 3 使用介绍

## 前言
关于 navigation 3 的诞生背景可以看[这篇文章](https://android-developers.googleblog.com/2025/05/announcing-jetpack-navigation-3-for-compose.html?m=1)。如果想看源码学习如何使用 Navigation 3 可以看这个 [repository](https://github.com/android/nav3-recipes)。

## 为什么要开发新的 navigation 3？
对于很多使用 Compose UI 的开发者来说，或许既有的 navigation 框架也许能满足他们的基本使用。但是当页面涉及到比较复杂的页面的时候就会有一些问题。举个例子假设你在开发一个**单页面应用/模块**，这时候如果有个需求要跳转到之前的某个页面，这时候路由的回退栈对你来说或许没那么友好，需要设置一堆属性，而且调试不是那么容易。而且这种方式很难满足越来越复杂的业务开发，所以这时候 navigation 3 就应运而生。

## navigation 3 的主要变化？
对于 navigation 3 最主要的变化是开发者获得了 `NavBackStack`的完整控制权，可以对路由操作进行高度的定制化操作。以前的回退栈只能通过被动观察，可能会导致路由跟当前存储的页面不匹配的情况。当然 navigation 3 的强大功能是一把双刃剑，使用不当开发起来肯定也是非常抓狂。所以对于大部分场景，我们可以使用默认的API来进行开发。

## navigation 3 的使用

### 简单使用方式

```kt
val backStack = remember { mutableStateListOf<Any>(RouteA) }

NavDisplay(
    backStack = backStack,
    onBack = { backStack.removeLastOrNull() },
    entryProvider = { key ->
        when (key) {
            is RouteA -> NavEntry(key) {
                ContentGreen("Welcome to Nav3") {
                    Button(onClick = {
                        backStack.add(RouteB("123"))
                    }) {
                        Text("Click to navigate")
                    }
                }
            }

            is RouteB -> NavEntry(key) {
                ContentBlue("Route id: ${key.id} ")
            }

            else -> {
                error("Unknown route: $key")
            }
        }
    }
)
```
只需要定义一个 mutableStateListOf 作为回退栈即可，或者更简单

```kt
    val backStack = rememberNavBackStack(RouteA)

    NavDisplay(
        backStack = backStack,
        onBack = { backStack.removeLastOrNull() },
        entryProvider = entryProvider {
            entry<RouteA> {
                ContentGreen("Welcome to Nav3") {
                    Button(onClick = {
                        backStack.add(RouteB("123"))
                    }) {
                        Text("Click to navigate")
                    }
                }
            }
            entry<RouteB> { key ->
                ContentBlue("Route id: ${key.id} ")
            }
        }
    )
}
```

### 自定义的使用方式

```kt
class TopLevelBackStack<T: Any>(startKey: T) {

    // Maintain a stack for each top level route
    private var topLevelStacks : LinkedHashMap<T, SnapshotStateList<T>> = linkedMapOf(
        startKey to mutableStateListOf(startKey)
    )

    // Expose the current top level route for consumers
    var topLevelKey by mutableStateOf(startKey)
        private set

    // Expose the back stack so it can be rendered by the NavDisplay
    val backStack = mutableStateListOf(startKey)

    private fun updateBackStack() =
        backStack.apply {
            clear()
            addAll(topLevelStacks.flatMap { it.value })
        }

    fun addTopLevel(key: T){

        // If the top level doesn't exist, add it
        if (topLevelStacks[key] == null){
            topLevelStacks.put(key, mutableStateListOf(key))
        } else {
            // Otherwise just move it to the end of the stacks
            topLevelStacks.apply {
                remove(key)?.let {
                    put(key, it)
                }
            }
        }
        topLevelKey = key
        updateBackStack()
    }

    fun add(key: T){
        topLevelStacks[topLevelKey]?.add(key)
        updateBackStack()
    }

    fun removeLast(){
        val removedKey = topLevelStacks[topLevelKey]?.removeLastOrNull()
        // If the removed key was a top level key, remove the associated top level stack
        topLevelStacks.remove(removedKey)
        topLevelKey = topLevelStacks.keys.last()
        updateBackStack()
    }
}
```
我们可以看到这个路由回退栈是高度自定义的，而且可以按照他的每个层级都可以自定义，这让我们开发复杂需求的时候可以更得心应手。

## NavDisplay 源码介绍
源码地址可见[这里](https://android.googlesource.com/platform/frameworks/support/+/44c07c80b0a8b6c357f95cf508264d0b4313fcb5/navigation3/navigation3/src/androidMain/kotlin/androidx/navigation3/NavDisplay.android.kt)


### 主要功能

1. **单窗格内容显示**：每次只显示一个导航目的地的内容
2. **自定义过渡动画**：支持自定义进入和退出过渡动画
3. **对话框支持**：能够将导航目的地显示为对话框
4. **后退处理**：集成了系统后退按钮的处理

### 源码介绍
```kotlin
@Composable
public fun <T : Any> NavDisplay(
    backstack: List<T>,
    modifier: Modifier = Modifier,
    wrapperManager: NavWrapperManager = rememberNavWrapperManager(emptyList()),
    contentAlignment: Alignment = Alignment.TopStart,
    sizeTransform: SizeTransform? = null,
    enterTransition: EnterTransition = fadeIn(...),
    exitTransition: ExitTransition = fadeOut(...),
    onBack: () -> Unit = { if (backstack is MutableList) backstack.removeAt(backstack.size - 1) },
    recordProvider: (key: T) -> NavRecord<out T>
)
```

1. **backstack**: 表示导航状态的键集合，不能为空
3. **wrapperManager**: 组合所有 NavContentWrapper 的管理器
5. **sizeTransform**: 用于控制大小变化的转换
6. **enterTransition**: 默认的进入过渡动画，默认为淡入效果
7. **exitTransition**: 默认的退出过渡动画，默认为淡出效果
8. **onBack**: 处理系统返回按钮的回调
9. **recordProvider**: 用于构造每个可能的 NavRecord 的 lambda 函数

## navigation 3 的 自适应布局介绍
### 核心组件:

-   `ListDetailSceneStrategy` Material 3 提供的场景策略，用于创建自适应的列表-详情布局


### 工作原理:

```kotlin
val backStack = rememberNavBackStack(ConversationList)
val listDetailStrategy = rememberListDetailSceneStrategy<NavKey>()

NavDisplay(
    backStack = backStack,
    onBack = { keysToRemove -> repeat(keysToRemove) { backStack.removeLastOrNull() } },
    sceneStrategy = listDetailStrategy,
    entryProvider = entryProvider {
        // 配置导航目的地
    }
)
```

### 关键特性:

1.  **三种窗格类型**:

    -   ` ListDetailSceneStrategy.listPane()` : 列表窗格，可以设置详情占位符

    -   ` ListDetailSceneStrategy.detailPane() `: 详情窗格

    -   `ListDetailSceneStrategy.extraPane() `: 额外窗格

1.  **自适应行为**:

    -   在窄屏设备上: 只显示一个窗格，用户需要导航来查看其他内容
    -   在宽屏设备上: 同时显示列表和详情窗格，提供分屏体验

1.  **实现示例**:

```kotlin
entry<ConversationList>(
    metadata = ListDetailSceneStrategy.listPane(
        detailPlaceholder = {
            ContentYellow("Choose a conversation from the list")
        }
    )
) {
    ContentRed("Welcome to Nav3") {
        Button(onClick = { backStack.add(ConversationDetail("ABC")) }) {
            Text("View conversation")
        }
    }
}
```

## 2. 自定义双窗格布局 (TwoPaneActivity)

这是一个展示如何创建自定义自适应布局的实现，使用 Scenes API 和自定义的`TwoPaneScene`和`TwoPaneSceneStrategy`。

### 核心组件:

-   `TwoPaneScene`: 自定义场景类，以 50/50 分割方式显示两个导航条目
-   `TwoPaneSceneStrategy`: 自定义场景策略，决定何时激活双窗格布局

### 工作原理:

```kotlin
val backStack = rememberNavBackStack(Home)
val twoPaneStrategy = remember { TwoPaneSceneStrategy<Any>() }

NavDisplay(
    backStack = backStack,
    onBack = { keysToRemove -> repeat(keysToRemove) { backStack.removeLastOrNull() } },
    sceneStrategy = twoPaneStrategy,
    entryProvider = entryProvider {
        // 配置导航目的地
    }
)
```

### 关键特性:

1.  **自定义布局逻辑**:

    ``` kotlin
    Row(modifier = Modifier.fillMaxSize()) {
        Column(modifier = Modifier.weight(0.5f)) {
            firstEntry.content.invoke(firstEntry.key)
        }
        Column(modifier = Modifier.weight(0.5f)) {
            secondEntry.content.invoke(secondEntry.key)
        }
    }
    ```

2.  **激活条件**:

    -   窗口宽度至少为 600dp (中等宽度断点)
    -   回退栈中最后两个条目都声明支持双窗格显示

3.  **窗口大小检测**:

    ```kotlin
    val windowSizeClass = currentWindowAdaptiveInfo().windowSizeClass
    if (!windowSizeClass.isWidthAtLeastBreakpoint(WIDTH_DP_MEDIUM_LOWER_BOUND)) {
        return null
    }
    ```

4.  **元数据标记**:

    ``` kotlin
    entry<Home>(metadata = TwoPaneScene.twoPane()) { ... }
    ```

## 自适应布局的共同特点

1.  **响应式设计**:

    -   根据屏幕尺寸自动调整布局
    -   在不同设备和方向上提供最佳用户体验

2.  **场景策略模式**:

    -   使用`SceneStrategy`接口来决定何时和如何显示特定布局
    -   通过`calculateScene`方法根据条件返回适当的场景

3.  **元数据驱动**:

    -   使用元数据来标记导航目的地的布局偏好
    -   允许每个目的地指定自己的显示方式

## 总结
本文中介绍 navigation 3 的一些用法，当然 navigation 3 目前还不是很稳定，希望感兴趣的可以尝试一下
