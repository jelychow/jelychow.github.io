---
theme: smartblue
excerpt: "逐步分析 Compose 重组要点，提升性能"
---
# Compose Performance Review

## Compose 性能提升的关键

很多初学者在刚接触 Compose 时可能会感到迷惑，不知道重组的重点在哪里。今天我们尝试从一个点出发，逐步分析如何编写高性能的 Compose UI。

关键词：**重组**。影响 Compose 性能最大的就是重组，所有的性能优化都应从重组开始分析。

### 为什么会重组？

我们先把复杂的问题简单化，然后逐步分析，再网络化记忆这些知识点。

1. 状态（state）变化导致重组
2. 可组合函数（composable function）的输入发生变化

上面两点写得很简单，没关系，我们一步步分析其中的道理。

## 如何降低状态变化导致的重组

### 状态变化导致重组的原理

具体原理见这篇[文章](https://jelychow.github.io/Jetpack-Compose-%E7%8A%B6%E6%80%81%E4%BE%9D%E8%B5%96%E8%BF%BD%E8%B8%AA%E5%8E%9F%E7%90%86%E8%AF%A6%E8%A7%A3/)

### 如何减少状态变化导致的重组？

#### 降低状态变化的频率

如何降低状态变化的频率呢？

1. 使用 Compose 的 `derivedStateOf` 来限制、降低状态变化的频率，以减少无效重组。
2. 使用 Kotlin Flow 的 `distinctUntilChanged` 来降低重复数据流。

#### 减少状态读取的范围

使用 defer read 的方式来降低状态影响的范围。所谓 defer read 就是通过将状态传入 lambda，然后传递给使用的地方，从而减少重组。为什么放到 lambda 里面可以降低重组？我们后面会详细讨论。

## 如何降低输入变化导致的重组？

先看一段代码：

```kotlin
@Composable
fun CustomText(
    text: String,
    modifier: Modifier = Modifier.padding(top = 46.dp),
) {
    Text(
        text = text,
        modifier = modifier.padding(32.dp),
        style = TextStyle(
            fontSize = 20.sp,
            textDecoration = TextDecoration.Underline,
            fontFamily = FontFamily.Monospace
        )
    )
}
```

```java
public static final void CustomText(@NotNull String text, @Nullable Modifier modifier, @Nullable Composer $composer, int $changed, int var4) {
   Intrinsics.checkNotNullParameter(text, "text");
   $composer = $composer.startRestartGroup(-837456306);
   ComposerKt.sourceInformation($composer, "C(CustomText)P(1)95@3325L244:Donout.kt#fvbk59");
   int $dirty = $changed;
   if ((var4 & 1) != 0) {
      $dirty = $changed | 6;
   } else if (($changed & 6) == 0) {
      $dirty = $changed | ($composer.changed(text) ? 4 : 2);
   }

   if ((var4 & 2) != 0) {
      $dirty |= 48;
   } else if (($changed & 48) == 0) {
      $dirty |= $composer.changed(modifier) ? 32 : 16;
   }

   if (($dirty & 19) == 18 && $composer.getSkipping()) {
      $composer.skipToGroupEnd();
   } else {
```

通过编译后的代码，我们可以看到 Compose 方法在进入重组时，会有一个标志字段 `$dirty` 通过位运算 **与** 来计算当前代码是否可以跳过。代码中有两处 `$composer.changed(text)` 和 `$composer.changed(modifier)`，通过 composer 调用 changed 方法来判断入参是否改变。

changed 方法源码在[这里](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/Composer.kt;drc=7b3f9a0bf941ce7d0ec219f2337cf0c944501f0b;bpv=1;bpt=1;l=2028)可见，感兴趣的可以看下。方法逻辑比较清晰，changed 方法有几个重要规则：

1. 对基本类型（Int, String 等）使用值相等比较
2. 对非稳定类型（如普通 class）使用引用相等比较
3. 对声明为 `@Stable` 的类或 data class 会启用结构相等比较

了解了规则后，我们的优化方向就明确了。对于基本类型，Compose 的编译器直接使用 equality，对于 `@Stable` 注解的类也会直接使用等值比较。除了 `@Stable` 和 `@Immutable` 注解的类型，Compose 里面有多少稳定类型？

- 所有基元值类型：`Boolean`、`Int`、`Long`、`Float`、`Char` 等。
- 字符串
- 所有函数类型（lambda）

当然 lambda 在没有开启强跳模式情况下，对于返回值不是 Unit 的类型，还是会被判断成不稳定类型，因为返回值可能会改变，从而导致重组。

了解了上面的稳定性规则后，我们可以在日常开发中尽量使用 Kotlin 的不可变类型作为入参，以减少重组次数。

## 使用 remember 系列减少重组的负载压力

在降低了重组次数后，还有没有其他方式能够降低重组的压力呢？Compose 提供了一个特别的操作——使用 `remember` 来减少重组过程的计算，避免多次无用的计算。

## 总结

回顾一下本篇内容，所有关于 Compose 的性能优化都是从重组开始分析的。一个是降低重组频率，重组频率从 state 和 stability 两个方面分析并进行优化；另一个是使用 Compose 里面的 `remember` 进行缓存，减少多次重复计算的压力。
