---
theme: fancy
highlight: arduino-light
excerpt: "深入解析Android应用的设计系统！探讨如何使用Material Design和Jetpack Compose创建一致、美观且高度可复用的UI组件库。"
---

## 准备工作

在学习本章之前个人建议观看一下官方文档中有关 compose theme 相关链接，有个大概的印象即可。我个人不喜欢在文章里面四处引用官方文档，这样于自己，别人都是无意义的事情，希望重点花在对项目的理解，知识应用上。
文档链接：

1.  [主题详解](https://developer.android.com/develop/ui/compose/designsystems/anatomy?hl=zh-cn)
2.  [主题自定义](https://developer.android.com/develop/ui/compose/designsystems/custom?hl=zh-cn)

## Design system 主要目的

在应用开发中 Design system 是一个非常见的概念。很多大公司都有自己的 design system，例如微软的 [FluentUi](https://developer.microsoft.com/en-us/fluentui#/)，就是一个横跨多端的 design system。通过应用 Design system 有效的保证了多端样式的统一，协作起来也更为方便，同时也更有利于设计师与程序员之间的沟通（因为所有的设计资源都是来自于现有的组件，所以开发者不用花很多时间去想着如何构思布局之类）。如果之前对于 design system 了解不多，那么我建议可以看看微软的 FluentUi，相信应该可以学到一些有用的东西。

## Now In Android 里面的 design system

### design system 示意图

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/88d2a9099b914b48bdd566f6836c4d62~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1741343358&x-orig-sign=rmvdfhLi5VtT%2FY%2FVF4QasbrdHr8%3D)

### design system 组成

一个 design system 主要由 components，icons，以及 theme 相关的文件组成。\
通常正经的公司都有一套自己的 design system，首先这个 design system 是夸端的，在设计稿，客户端，web上面会有一些开发用到的 components，icons，theme 相关的页面。在设计和开发过程中都会按照 design system 上面的组件，theme 进行设计，这样能够最大程度的保证三端的稳定性，用工程的手段保证项目的 UI 的一致性。

#### 举个例子

先看一下 material design 的 demo 设计图

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/8a26604743c84258800824c703ce298d~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1741343358&x-orig-sign=VBu5EVhuhiExQtWgPQ84YuHNcGg%3D)

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/e53f71cb16f54533afd1a99ea656a72e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1741343358&x-orig-sign=CjInsagmuk9Bz9K88phCWv51FDw%3D)

上面的设计图是一个按钮组件，边上是这个按钮的属性，我们可以看到选用的是 M3 design kit 里面的 Button 这个组件，他的 style 是 filled, 他的 state 是 enabled，show icon 是 off，label text 是 Download。在正常的开发流程里面我们的组件库库里面应该有这个 Button 的组件，并且 style 文件里面应该有 filled 这个样式，Button 组件应该支持 enabled 属性，支持展示隐藏 icon，并且支持标签展示。在这里就会有好奇宝宝提问了，我 iOS 不支持咋办，我 web button 没有 icon 功能咋办，这里通常来说需要每一端（尤其是设计端）去补上这个 component。这样的方式看起来前期会很麻烦，因为一个新项目会需要创建很多组件，样式，颜色等等。但是他在工程管理方面其实是走在前面的，因为你所有的工作都不会重复，后面如果再次出现这样的组件，你只需要从组件库里面 copy\&paste 即可。这种方式也保证 UI 的高还原度，因为所有的组件，样式都是预定义的，开发者不需要花很多心思在 UI 实现上，重点在实现 feature 逻辑上即可。

### theme **系统的命名习惯**

首先来看一段 Now In Android 里面的 colorschema 命名，这是一个容易被很多人忽略的细节，个人建议如果没有接触过的一定要多理解学习。

```kt
// 颜色定义
internal val Blue10 = Color(0xFF001F28)
internal val Blue20 = Color(0xFF003544)
internal val Blue30 = Color(0xFF004D61)
internal val Blue40 = Color(0xFF006780)
internal val Blue80 = Color(0xFF5DD5FC)
internal val Blue90 = Color(0xFFB8EAFF)
internal val DarkGreen10 = Color(0xFF0D1F12)
internal val DarkGreen20 = Color(0xFF223526)
internal val DarkGreen30 = Color(0xFF394B3C)
internal val DarkGreen40 = Color(0xFF4F6352)
internal val DarkGreen80 = Color(0xFFB7CCB8)
...
// 样式定义
val DarkAndroidColorScheme = darkColorScheme(
    primary = Green80,
    onPrimary = Green20,
    primaryContainer = Green30,
    onPrimaryContainer = Green90,
    secondary = DarkGreen80,
    onSecondary = DarkGreen20,
    secondaryContainer = DarkGreen30,
    onSecondaryContainer = DarkGreen90,
    tertiary = Teal80,
    ...
)
// 使用
NotificationDot(
            color = MaterialTheme.colorScheme.tertiary,
            modifier = Modifier.size(8.dp),
        )
```

很多同学是不是有疑惑，这样写是不是🖌️🐍🍩⚽，明明可以这样写的 MaterialTheme.colorScheme.DarkGreen10 为什么还要再多此一举？ 咋一听好像有点道理，但是如果我要支持 light mode 呢，该如何是好呢？总不能使用 if else 来管理样式吧。其实行业内对于组件样式的命名早就有了一套成熟的办法了，现在业界普遍的方式叫做 design token。\
下面看下 [material design 的介绍](https://m3.material.io/foundations/design-tokens/overview)

```kt
-   Tokens point to style values like colors, fonts, and measurements
-   Use design tokens instead of hardcoded values
-   Each token is named for how or where it’s used (for example, **md.fab.container.color** sets the container color for a FAB)
-   Even if a token’s end value is changed, its name and use remain the same
-  ...
```

同时也推荐一些其他链接希望大家从中学习，[link1](https://www.uisdc.com/design-token)，[link2](https://semi.design/zh-CN/basic/tokens)
总结出来 design token 这种命名方式以使用方式或者使用地点命名，而不是直接 hardcode，这样方式更符合现代方案。不知道讲了这么多，你有没有对 token design 这种现在主流的设计方案有所了解，如果有疑惑的地方，欢迎留言讨论。

## CompositionLocal

之所以要单独开一节来讲 CompositionLocal 和 Theme是因为涉及到知识点有点多，而且这对于开发者来说又是一项不可或缺的知识。想要了解 Theme 那么必须了解 CompositionLocal 相关知识。

### compose，composable，composition，compositionLocal 傻傻分不清楚？

我之所以抛出这个问题是为了先给大家对 compose 有一个整体的印象，形成一个系统的概念。

*   **compose**：整体框架，用于构建 UI。

*   **composable**：标记函数，使其成为可组合的 UI 组件。（形容词当注解）

*   **composition**：UI 组件的动态树结构，负责管理 UI 的更新。（compose 是个动词，执行完的结果，一个树形结构）

*   **compositionLocal**：用于在组件树中共享数据的机制。

### compositionLocal 设计的目的是什么？

由于 compose 大量借鉴（抄袭）了 react 的方案，在这里我推荐大家看一下 react 的[文档](https://www.reactjs.cn/learn/passing-data-deeply-with-context#step-3-provide-the-context)，总结起来就是为了解决层级过深引起的传值问题，由于 compose ui 推荐的方式**显示传值**，这种方法在单页面应用，以及其他复杂页面会变得十分复杂，维护样式也会变得异常复杂，于是 compositionLocal 应运而生。

### 如何快速理解 CompositionLocal？

相信很多人和我一样看到 CompositionLocal 这个概念的时候会有疑问，为啥命名叫 compositionLocal？很多框架里面的命名都是约定俗成的，很多看着平平无奇的代码，都是无数次工程演进的结果。现在我给大家介绍一个老朋友 ThreadLocal，大家比照 ThreadLocal 来理解 compositionLocal 是不是要容易的多？很容易可以知道 compositionLocal 也是用来保存 composition 的本地变量，那么是不是也有一个 CompositionLocalMap 这样的东西存在？那自然也是存在的。CompositionLocalMap 是用来保存 CompositionLocal ，使用 CompositionLocal\<Any?> 为 key，他是基于 TrieNode 实现的一种数据结构，极大的降低了树的搜索复杂度。关于 Trie（前缀树）的介绍就不在这里仔细展开了，有兴趣的同学可以去 [leetcode](https://leetcode.cn/problems/implement-trie-prefix-tree/description/) 尝试自己实现一下，很容易实现的 😄。

### 如何使用 CompositionLocal？

CompositionLocal 使用跟 React 里面的 context 非常像，分三步走。

1.  创建 CompositionLocal
2.  使用 CompositionLocalProvider 为 CompositionLocal 提供值
3.  使用 CompositionLocal
    大家可以简单的理解为：创建，初始化，使用。下面用 Now In Android 代码展示一下

```kt
// step 1
val LocalAnalyticsHelper = staticCompositionLocalOf<AnalyticsHelper> {
    // Provide a default AnalyticsHelper which does nothing. This is so that tests and previews
    // do not have to provide one. For real app builds provide a different implementation.
    NoOpAnalyticsHelper()
}

// step 2
CompositionLocalProvider(
    LocalAnalyticsHelper provides analyticsHelper,
    LocalTimeZone provides currentTimeZone,
) {
    NiaTheme(
        darkTheme = themeSettings.darkTheme,
        androidTheme = themeSettings.androidTheme,
        disableDynamicTheming = themeSettings.disableDynamicTheming,
    ) {
        NiaApp(appState)
    }
}

// step 3
val analyticsHelper = LocalAnalyticsHelper.current
```

这里面有一个细节，CompositionLocal.current 是获取离他最近的的 CompositionLocalProvider 提供的 CompositionLocal。这意味着 CompositionLocal 实际上是可以存在多份的，一个树结构中可能存在多个 CompositionLocal 实例，散布在不同节点上。举个例子本例中 CompositionLocalProvider 提供了一个默认的 staticCompositionLocalOf<AnalyticsHelper>，但是子组件完全有可能自己提供一个新的 staticCompositionLocalOf<AnalyticsHelper>，这种做法是完全可能的。不过一般不建议这么做，这么操作可能会使得代码维护变得困难。

### Task：compositionLocalOf 与 staticCompositionLocalOf 有什么区别？他们各有什么使用场景？

这算是留给大家的一个小任务，希望你可以先自己思考一下，然后再跟官方文档对比一下，看看自己理解是否正确。

## Theme

### Theme 的概念

> 一个主题通常由许多子系统组成，这些子系统用于对常见的视觉概念和行为概念进行分组，可以使用具有主题值的类进行建模

### 举例说明

下面使用系统自带 MaterialTheme 来做举例说明

```kt
    @Composable
fun MaterialTheme(
    colorScheme: ColorScheme = MaterialTheme.colorScheme,
    shapes: Shapes = MaterialTheme.shapes,
    typography: Typography = MaterialTheme.typography,
    content: @Composable () -> Unit
) {
    val rememberedColorScheme =
        remember {colorScheme.copy()}
            .apply { updateColorSchemeFrom(colorScheme) }
    val selectionColors = rememberTextSelectionColors(rememberedColorScheme)
    CompositionLocalProvider(
        LocalColorScheme provides rememberedColorScheme,
        LocalShapes provides shapes,
        LocalTextSelectionColors provides selectionColors,
        LocalTypography provides typography
    ) {
        ProvideTextStyle(value = typography.bodyLarge, content = content)
    }
}

object MaterialTheme {
    val colorScheme: ColorScheme
        @Composable
        @ReadOnlyComposable
        @SuppressWarnings("HiddenTypeParameter", "UnavailableSymbol")
        get() = LocalColorScheme.current

    val typography: Typography
        @Composable @ReadOnlyComposable get() = LocalTypography.current

    val shapes: Shapes
        @Composable @ReadOnlyComposable get() = LocalShapes.current
}
    
```

通过上面的 MaterialTheme 代码我们可以看到他的主题主要有 颜色，形状，排版 三个子系统组成，在主题 fun 里面使用 compositionLocal 的将这些子系统一起提供给作用域里面的 compose。一般来说 Theme 都会通过 object 导出样式方便使用方调用， 当然 compose 的主题灵活度很高，你完全可以添加很多自定义主题，说到底 compose 的 theme 其实是一组 compositionLocals。你可以按照项目需要灵活的定制化你的主题，比如说 colorscheme，你完全可以添加一个新的子系统自己命名叫 fancy, 然后你自己定义 compositionLocal，provide 给使用方即可。

### Theme 使用步骤

一般来说 Theme 定义遵循下面 3 个主要步骤

1.  定义子系统
2.  使用 Theme 方法组装子系统
3.  使用 Theme object 导出子系统

### Now In Android 中 Theme 是如何设计的？

由于仓库里面的代码较长，我不好直接 copy 过来，影响阅读体验，我在这里放一个[链接](https://github.com/android/nowinandroid/blob/main/core/designsystem/src/main/kotlin/com/google/samples/apps/nowinandroid/core/designsystem/theme/Theme.kt)，可以点击进去查看一下他的 Theme 设计。\
Now In Android 并没有使用 object 导出 Theme，这一点跟官方文档推荐的不太一样，我个人是不建议这样写的，这样写会使得很多子系统应用很奇怪，让使用方很迷惑，下面我放代码库中的一段代码：

```kt
fun NiaGradientBackground(
   modifier: Modifier = Modifier,
   gradientColors: GradientColors = LocalGradientColors.current,
   content: @Composable () -> Unit,
)
```

通过上面的代码我们看到使用方直接调用了子系统，而不是通过 Theme.gradient 类似的方式来实现，这种直接向使用方暴露子系统的方式，着实会让调用方摸不着头脑。开发者应该把重点放在 Theme 上，如果这时候突然出现一个子系统散落在四处无疑会给阅读还有使用都带来负担。当然这也破坏了封装，我个人对此做了一些修改。代码链接[传送地址](https://github.com/android/nowinandroid/pull/1808/files#diff-a72250a640d53322e5eb10ff8cd0d4db01752b81687cda0daff3e5693b3e78b6)，欢迎分享交流你对 theme 的见解。

## 总结

*   compositionLocal 是为了解决层级过长，传递参数不易而产生的一种状态共享方案。
*   Theme 是一组 compositionLocal 的组合
*   本文还介绍了如何定义和使用 compositionLocal，Theme

## 本章完。谢谢观看
