---
title: '使用工厂模式和接口封装实验性API的好处'
date: 2025-07-31
permalink: /posts/encapsulating_experimental_apis/
tags:
  - API Design
  - Factory Pattern
  - Kotlin
  - Best Practices
---

<div class="language-selector">
  <span class="lang-link active">中文</span> | 
  <a href="/posts/encapsulating_experimental_apis/en/" class="lang-link">English</a>
</div>

# 使用工厂模式和接口封装实验性API的好处

最近在一个多平台项目中，我遇到了一个让人头疼的问题：项目中大量使用了标记为"实验性"的API，导致代码库中充斥着各种 `@OptIn` 注解。每次升级依赖库，总会有几个API变动，需要修改大量代码。这种情况下，我开始思考如何更优雅地处理这些实验性API，最终通过工厂模式和接口封装找到了解决方案。今天就来分享一下这个实践经验。

### 实验性API到底有什么问题？

在深入解决方案之前，先聊聊为什么实验性API会成为一个问题。以Kotlin的`kotlinx-datetime`库为例，使用它时我们会遇到这些麻烦：

1. **注解污染**：代码中到处都是`@OptIn(ExperimentalTime::class)`，看着就烦
2. **编译器唠叨**：忘记加注解？编译器立马警告或报错
3. **不稳定性**：实验性API随时可能变化，一个小版本升级可能导致大量代码需要修改
4. **耦合度高**：业务逻辑直接依赖实验性API，导致后续难以替换或升级
5. **测试困难**：直接使用实验性API的代码往往难以进行单元测试

这些问题在小项目中可能不明显，但在我们的多平台项目中，随着代码库增长，这些问题逐渐放大，最终变成了一个不得不解决的技术债。

### 封装的核心思想

解决这个问题的关键在于"隔离变化"。我们需要将那些可能变化的实验性API隔离到一个地方，而不是散布在整个代码库中。具体做法是：

1. 定义稳定的接口，表达我们真正需要的功能
2. 在单一实现类中使用实验性API
3. 通过工厂模式提供接口实例

这样，当实验性API变化时，我们只需修改实现类，而不必触碰业务代码。

### 实战案例：封装时间相关API

下面是我在项目中实际使用的例子，展示如何封装`kotlinx-datetime`的实验性API：

#### 1. 定义稳定接口

首先，我们需要定义一个清晰的接口，只包含我们真正需要的功能：

```kotlin
/**
 * 时间提供者接口
 * 封装时间相关操作，避免直接使用实验性的时间API
 */
interface TimeProvider {
    /**
     * 获取当前时间的毫秒时间戳
     */
    fun getCurrentTimeMillis(): Long

    /**
     * 格式化时间戳为日期时间字符串
     * 格式：yyyy-MM-dd HH:mm
     */
    fun formatDateTime(timestamp: Long): String

    /**
     * 格式化时间戳为日期字符串
     * 格式：yyyy年MM月dd日
     */
    fun formatDate(timestamp: Long): String

    /**
     * 获取当前系统默认时区
     */
    fun getCurrentTimeZoneId(): String

    /**
     * 判断时间戳是否已过期
     * @param timestamp 过期时间的时间戳（毫秒）
     * @param bufferMillis 缓冲时间（毫秒），默认为5000毫秒
     */
    fun isExpired(timestamp: Long, bufferMillis: Long = 5000): Boolean
}
```

这个接口设计得很干净，没有暴露任何实验性API的细节，只关注我们的业务需求。

#### 2. 实现类：所有实验性API的"集中营"

接下来，创建一个实现类，所有的实验性API使用都被限制在这个类中：

```kotlin
@file:OptIn(ExperimentalTime::class)

/**
 * TimeProvider的默认实现
 * 使用kotlinx-datetime库实现时间相关功能
 */
class DefaultTimeProvider : TimeProvider {

    override fun getCurrentTimeMillis(): Long {
        return Clock.System.now().toEpochMilliseconds()
    }

    override fun formatDateTime(timestamp: Long): String {
        val instant = Instant.fromEpochMilliseconds(timestamp)
        val localDateTime = instant.toLocalDateTime(TimeZone.currentSystemDefault())
        return "${localDateTime.year}-${localDateTime.monthNumber.toString().padStart(2, '0')}-${localDateTime.dayOfMonth.toString().padStart(2, '0')} " +
                "${localDateTime.hour.toString().padStart(2, '0')}:${localDateTime.minute.toString().padStart(2, '0')}"
    }

    override fun formatDate(timestamp: Long): String {
        val instant = Instant.fromEpochMilliseconds(timestamp)
        val localDateTime = instant.toLocalDateTime(TimeZone.currentSystemDefault())
        return "${localDateTime.year}年${localDateTime.monthNumber}月${localDateTime.dayOfMonth}日"
    }

    override fun getCurrentTimeZoneId(): String {
        return TimeZone.currentSystemDefault().id
    }

    override fun isExpired(timestamp: Long, bufferMillis: Long): Boolean {
        return Clock.System.now().toEpochMilliseconds() > (timestamp - bufferMillis)
    }
}
```

注意，所有实验性API的使用都被限制在这个单一的文件中，`@OptIn`注解只需要在文件级别添加一次。

#### 3. 工厂：提供接口实例

最后，创建一个工厂类来管理实例：

```kotlin

/**
 * 时间提供者工厂
 * 用于获取TimeProvider实例
 */
object TimeProviderFactory {
    // 默认实例，可以在测试中替换
    private var instance: TimeProvider = DefaultTimeProvider()

    /**
     * 获取TimeProvider实例
     */
    fun getInstance(): TimeProvider {
        return instance
    }

    /**
     * 设置自定义TimeProvider实例
     * 主要用于测试
     */
    fun setInstance(customInstance: TimeProvider) {
        instance = customInstance
    }
}
```

### 在代码中使用这种模式

现在，我们可以在代码中使用工厂来获取TimeProvider实例，而不是直接使用实验性API：

```kotlin
class UserViewModel(
    private val repository: UserRepository,
    // 依赖可以注入，也可以从工厂获取
    private val timeProvider: TimeProvider = TimeProviderFactory.getInstance()
) : ViewModel() {
    
    fun formatLastLoginTime(timestamp: Long): String {
        return timeProvider.formatDateTime(timestamp)
    }
    
    fun checkSessionExpired(sessionExpiryTime: Long): Boolean {
        // 这里不需要任何实验性API注解！
        return timeProvider.isExpired(sessionExpiryTime)
    }
}
```

业务代码现在变得干净，不需要任何实验性API注解。

### 测试变得简单

这种模式的一个最大好处是简化了测试。我们可以轻松创建一个模拟实现：

```kotlin
class MockTimeProvider : TimeProvider {
    var currentTimeMillis = 0L
    
    override fun getCurrentTimeMillis(): Long = currentTimeMillis
    
    // 其他方法的简单实现...
}
```

通过工厂的`setInstance`方法，我们可以轻松替换实现，实现对时间的完全控制，测试变得简单而可靠。

### 升级依赖时的惊喜

这种模式的价值在我们升级依赖时体现得最为明显。记得有一次，我们将`kotlinx-datetime`从0.2.1升级到0.3.0，API有一些变化。在以前，这意味着要修改散布在各处的代码；而现在，我们只需要更新`DefaultTimeProvider`一个类，其他代码完全不受影响。

整个升级过程从原来的"噩梦"变成了"小菜一碟"，这种感觉真的很爽！

### 总结：值得推广的最佳实践

通过实际项目经验，我认为这种封装实验性API的模式值得在更多项目中推广：

1. **隔离变化**：将不稳定的API限制在单一实现类中
2. **接口稳定**：为业务代码提供稳定的接口
3. **测试友好**：便于在测试中替换实现
4. **升级平滑**：依赖升级时只需修改实现类
5. **代码整洁**：业务代码中没有实验性API的痕迹

这种模式不仅适用于时间相关API，也适用于任何标记为实验性或不稳定的API。如果你的项目中也面临类似的问题，不妨试试这种方法，相信会有不错的效果。

你有没有遇到过类似的问题？或者有其他处理实验性API的方法？欢迎在评论区分享你的经验！
