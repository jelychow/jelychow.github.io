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
  <button id="lang-zh" class="lang-btn" onclick="switchLanguage('zh')">中文</button>
  <button id="lang-en" class="lang-btn active" onclick="switchLanguage('en')">English</button>
</div>

<div id="content-zh" class="lang-content">
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
package com.questiontop.questions.util.time

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

package com.questiontop.questions.util.time

import kotlinx.datetime.TimeZone
import kotlinx.datetime.toLocalDateTime
import kotlin.time.Clock
import kotlin.time.ExperimentalTime
import kotlin.time.Instant

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
        return "${localDateTime.year}-${localDateTime.monthNumber.toString().padStart(2, '0')}-" +
            "${localDateTime.dayOfMonth.toString().padStart(2, '0')} " +
            "${localDateTime.hour.toString().padStart(2, '0')}:" +
            localDateTime.minute.toString().padStart(2, '0')
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
        val currentTimeMillis = getCurrentTimeMillis()
        return currentTimeMillis >= (timestamp - bufferMillis)
    }
}
```

注意这个类的特点：
- `@OptIn`注解只出现在文件顶部一次
- 所有实验性API的使用都被限制在这个类中
- 对外暴露的都是稳定的接口方法

#### 3. 工厂：提供统一访问点

最后，创建一个工厂类来管理实例：

```kotlin
package com.questiontop.questions.util.time

/**
 * 时间提供者工厂
 * 用于获取TimeProvider实例
 */
object TimeProviderFactory {
    private var instance: TimeProvider? = null

    /**
     * 获取TimeProvider实例
     * 如果实例不存在，创建一个新的DefaultTimeProvider实例
     */
    fun getInstance(): TimeProvider {
        if (instance == null) {
            instance = DefaultTimeProvider()
        }
        return instance!!
    }

    /**
     * 设置TimeProvider实例
     * 用于测试或自定义实现
     */
    fun setInstance(timeProvider: TimeProvider) {
        instance = timeProvider
    }
}
```

工厂模式在这里提供了几个好处：
- 单例管理，避免重复创建实例
- 提供替换实现的能力，特别是在测试时
- 为所有客户端代码提供统一的访问点

### 实际使用体验

这套模式在我们的项目中应用后，带来了明显的改善。下面是一些实际的使用场景：

#### 在ViewModel中使用

```kotlin
class InterviewViewModel(
    private val timeProvider: TimeProvider = TimeProviderFactory.getInstance()
) {
    fun formatInterviewTime(timestamp: Long): String {
        return timeProvider.formatDateTime(timestamp)
    }
    
    fun isInterviewExpired(expiresAt: Long): Boolean {
        return timeProvider.isExpired(expiresAt)
    }
}
```

看，ViewModel中完全没有实验性API的痕迹，代码干净整洁。

#### 令牌刷新服务

```kotlin
class KtorTokenRefreshService(
    private val httpClient: HttpClient,
    private val timeProvider: TimeProvider = TimeProviderFactory.getInstance(),
) : TokenRefreshService {
    override suspend fun refreshToken(refreshToken: String?): Token {
        // 使用httpClient获取新token
        val authResult = httpClient.post("auth/refresh") {
            setBody(mapOf("refresh_token" to refreshToken))
        }.body<AuthResponse>()
        
        // 计算过期时间
        val currentTimeMillis = timeProvider.getCurrentTimeMillis()
        val expirationTimeMillis = currentTimeMillis + (authResult.data!!.expiresAt * 1000L)

        return Token(
            accessToken = authResult.data.accessToken,
            refreshToken = authResult.data.refreshToken,
            expiresAt = expirationTimeMillis,
        )
    }
}
```

在这个服务中，我们使用`timeProvider`来处理时间相关的逻辑，而不直接依赖实验性API。

#### 在UI层使用

```kotlin
// 在Compose组件中
@Composable
private fun EventTimeDisplay(timestamp: Long) {
    val timeProvider = remember { TimeProviderFactory.getInstance() }
    
    Text(
        text = try {
            timeProvider.formatDate(timestamp)
        } catch (e: Exception) {
            "未知日期"
        },
        style = MaterialTheme.typography.bodyMedium
    )
}
```

UI层也是如此，干净利落地使用`timeProvider`，没有实验性API的痕迹。

### 测试变得超级简单

这种封装方式的一个巨大好处是测试变得异常简单。看看这个例子：

```kotlin
class TokenServiceTest {
    private val mockTimeProvider = MockTimeProvider()
    private lateinit var tokenService: TokenRefreshService
    
    @Before
    fun setup() {
        // 替换默认实现为测试用的mock
        TimeProviderFactory.setInstance(mockTimeProvider)
        tokenService = TokenRefreshService(mockHttpClient)
    }
    
    @Test
    fun `token should be expired after expiration time`() {
        // 设置当前时间
        mockTimeProvider.currentTimeMillis = 1625097600000L // 2021-07-01 00:00:00
        
        // 创建一个1小时后过期的token
        val token = Token(
            accessToken = "test",
            refreshToken = "test",
            expiresAt = mockTimeProvider.currentTimeMillis + 3600000
        )
        
        // 验证token未过期
        assertFalse(tokenService.isTokenExpired(token))
        
        // 前进时间1小时零1分钟
        mockTimeProvider.currentTimeMillis += 3660000
        
        // 验证token已过期
        assertTrue(tokenService.isTokenExpired(token))
    }
}

// 测试用的Mock实现
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
</div>

<div id="content-en" class="lang-content active">
# Encapsulating Experimental APIs with Factory Pattern and Interfaces

Recently, I ran into a frustrating problem in a multiplatform project: our codebase was littered with `@OptIn` annotations due to heavy use of experimental APIs. Every dependency upgrade meant changing code in dozens of places. After some headaches, I found a cleaner solution using the Factory Pattern and interfaces to encapsulate these experimental APIs. Let me share this approach that saved our project from annotation hell.

### The Problem with Experimental APIs

Before diving into the solution, let's talk about why experimental APIs are problematic in the first place. Taking Kotlin's `kotlinx-datetime` library as an example:

1. **Annotation Pollution**: Code filled with `@OptIn(ExperimentalTime::class)` annotations everywhere
2. **Compiler Warnings**: Forget an annotation? Get ready for warnings or errors
3. **API Instability**: Experimental APIs can change between versions, breaking your code
4. **Tight Coupling**: Business logic directly depends on unstable APIs
5. **Testing Headaches**: Code using experimental APIs directly is often harder to test

These issues might seem minor in small projects, but they compound quickly in larger codebases. In our multiplatform project, this became a significant pain point that we couldn't ignore anymore.

### The Core Idea: Isolation

The key insight is to "isolate change." We need to contain potentially changing experimental APIs in one place, rather than spreading them throughout the codebase. The approach is:

1. Define stable interfaces that express what we actually need
2. Use experimental APIs in a single implementation class
3. Provide interface instances through a factory pattern

This way, when experimental APIs change, we only need to modify the implementation class without touching business code.

### Real-World Example: Encapsulating Time APIs

Here's the actual implementation I used in our project to encapsulate `kotlinx-datetime` experimental APIs:

#### 1. The Interface

First, we define a clean interface that only includes what we really need:

```kotlin
package com.questiontop.questions.util.time

/**
 * Time provider interface
 * Encapsulates time-related operations to avoid direct use of experimental time APIs
 */
interface TimeProvider {
    /**
     * Get current time in milliseconds
     */
    fun getCurrentTimeMillis(): Long

    /**
     * Format timestamp to date-time string
     * Format: yyyy-MM-dd HH:mm
     */
    fun formatDateTime(timestamp: Long): String

    /**
     * Format timestamp to date string
     * Format: yyyy年MM月dd日
     */
    fun formatDate(timestamp: Long): String

    /**
     * Get current system default timezone ID
     */
    fun getCurrentTimeZoneId(): String

    /**
     * Check if a timestamp is expired
     * @param timestamp Expiration timestamp (milliseconds)
     * @param bufferMillis Buffer time (milliseconds), default is 5000 milliseconds
     */
    fun isExpired(timestamp: Long, bufferMillis: Long = 5000): Boolean
}
```

Notice how this interface is completely clean - no experimental API details exposed, just focusing on our business needs.

#### 2. The Implementation: Where All Experimental APIs Live

Next, create an implementation class where all experimental API usage is contained:

```kotlin
@file:OptIn(ExperimentalTime::class)

package com.questiontop.questions.util.time

import kotlinx.datetime.TimeZone
import kotlinx.datetime.toLocalDateTime
import kotlin.time.Clock
import kotlin.time.ExperimentalTime
import kotlin.time.Instant

/**
 * Default implementation of TimeProvider
 * Uses kotlinx-datetime library to implement time-related functions
 */
class DefaultTimeProvider : TimeProvider {

    override fun getCurrentTimeMillis(): Long {
        return Clock.System.now().toEpochMilliseconds()
    }

    override fun formatDateTime(timestamp: Long): String {
        val instant = Instant.fromEpochMilliseconds(timestamp)
        val localDateTime = instant.toLocalDateTime(TimeZone.currentSystemDefault())
        return "${localDateTime.year}-${localDateTime.monthNumber.toString().padStart(2, '0')}-" +
            "${localDateTime.dayOfMonth.toString().padStart(2, '0')} " +
            "${localDateTime.hour.toString().padStart(2, '0')}:" +
            localDateTime.minute.toString().padStart(2, '0')
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
        val currentTimeMillis = getCurrentTimeMillis()
        return currentTimeMillis >= (timestamp - bufferMillis)
    }
}
```

Note the key characteristics:
- The `@OptIn` annotation appears only once at the file level
- All experimental API usage is contained in this single class
- Only stable interface methods are exposed to the outside world

#### 3. The Factory: Providing a Unified Access Point

Finally, create a factory to manage instances:

```kotlin
package com.questiontop.questions.util.time

/**
 * Time provider factory
 * Used to get TimeProvider instances
 */
object TimeProviderFactory {
    private var instance: TimeProvider? = null

    /**
     * Get TimeProvider instance
     * If instance doesn't exist, creates a new DefaultTimeProvider instance
     */
    fun getInstance(): TimeProvider {
        if (instance == null) {
            instance = DefaultTimeProvider()
        }
        return instance!!
    }

    /**
     * Set TimeProvider instance
     * Used for testing or custom implementations
     */
    fun setInstance(timeProvider: TimeProvider) {
        instance = timeProvider
    }
}
```

The factory pattern provides several benefits here:
- Singleton management to avoid creating multiple instances
- Ability to swap implementations, especially for testing
- A unified access point for all client code

### Real Usage Experience

After applying this pattern in our project, we saw significant improvements. Here are some real-world usage examples:

#### In ViewModels

```kotlin
class InterviewViewModel(
    private val timeProvider: TimeProvider = TimeProviderFactory.getInstance()
) {
    fun formatInterviewTime(timestamp: Long): String {
        return timeProvider.formatDateTime(timestamp)
    }
    
    fun isInterviewExpired(expiresAt: Long): Boolean {
        return timeProvider.isExpired(expiresAt)
    }
}
```

See how clean the ViewModel is? No trace of experimental APIs anywhere.

#### Token Refresh Service

```kotlin
class KtorTokenRefreshService(
    private val httpClient: HttpClient,
    private val timeProvider: TimeProvider = TimeProviderFactory.getInstance(),
) : TokenRefreshService {
    override suspend fun refreshToken(refreshToken: String?): Token {
        // Get new token using httpClient
        val authResult = httpClient.post("auth/refresh") {
            setBody(mapOf("refresh_token" to refreshToken))
        }.body<AuthResponse>()
        
        // Calculate expiration time
        val currentTimeMillis = timeProvider.getCurrentTimeMillis()
        val expirationTimeMillis = currentTimeMillis + (authResult.data!!.expiresAt * 1000L)

        return Token(
            accessToken = authResult.data.accessToken,
            refreshToken = authResult.data.refreshToken,
            expiresAt = expirationTimeMillis,
        )
    }
}
```

In this service, we use `timeProvider` for time-related logic instead of directly depending on experimental APIs.

#### In UI Components

```kotlin
// In a Composable function
@Composable
private fun EventTimeDisplay(timestamp: Long) {
    val timeProvider = remember { TimeProviderFactory.getInstance() }
    
    Text(
        text = try {
            timeProvider.formatDate(timestamp)
        } catch (e: Exception) {
            "Unknown Date"
        },
        style = MaterialTheme.typography.bodyMedium
    )
}
```

The UI layer is equally clean, using `timeProvider` without any experimental API traces.

### Testing Becomes a Breeze

One huge benefit of this approach is how much easier testing becomes. Check out this example:

```kotlin
class TokenServiceTest {
    private val mockTimeProvider = MockTimeProvider()
    private lateinit var tokenService: TokenRefreshService
    
    @Before
    fun setup() {
        // Replace default implementation with test mock
        TimeProviderFactory.setInstance(mockTimeProvider)
        tokenService = TokenRefreshService(mockHttpClient)
    }
    
    @Test
    fun `token should be expired after expiration time`() {
        // Set current time
        mockTimeProvider.currentTimeMillis = 1625097600000L // 2021-07-01 00:00:00
        
        // Create token that expires in 1 hour
        val token = Token(
            accessToken = "test",
            refreshToken = "test",
            expiresAt = mockTimeProvider.currentTimeMillis + 3600000
        )
        
        // Verify token is not expired
        assertFalse(tokenService.isTokenExpired(token))
        
        // Advance time by 1 hour and 1 minute
        mockTimeProvider.currentTimeMillis += 3660000
        
        // Verify token is now expired
        assertTrue(tokenService.isTokenExpired(token))
    }
}

// Mock implementation for testing
class MockTimeProvider : TimeProvider {
    var currentTimeMillis = 0L
    
    override fun getCurrentTimeMillis(): Long = currentTimeMillis
    
    // Simple implementations of other methods...
}
```

Through the factory's `setInstance` method, we can easily swap implementations, gaining complete control over time for testing purposes.

### The Upgrade Surprise

The value of this pattern became most evident when upgrading dependencies. I remember when we upgraded `kotlinx-datetime` from 0.2.1 to 0.3.0, which had some API changes. Previously, this would have meant modifying code scattered throughout the codebase; now, we only needed to update the `DefaultTimeProvider` class, leaving all other code untouched.

The entire upgrade process went from "nightmare" to "piece of cake" - that feeling was priceless!

### Conclusion: A Best Practice Worth Spreading

Based on real project experience, I believe this pattern of encapsulating experimental APIs is worth adopting in more projects:

1. **Isolate Change**: Contain unstable APIs in a single implementation class
2. **Stable Interfaces**: Provide stable interfaces for business code
3. **Testing Friendly**: Easy to swap implementations for testing
4. **Smooth Upgrades**: Only need to modify the implementation class when dependencies change
5. **Clean Code**: No experimental API traces in business code

This pattern isn't just for time-related APIs - it works for any API marked as experimental or unstable. If you're facing similar issues in your project, give this approach a try. I think you'll be pleasantly surprised by the results.

Have you encountered similar problems? Or do you have other approaches to handling experimental APIs? I'd love to hear your experiences in the comments!
</div>

<style>
.language-selector {
  display: flex;
  margin-bottom: 20px;
}

.lang-btn {
  padding: 8px 16px;
  margin-right: 10px;
  background-color: #f0f0f0;
  border: 1px solid #ddd;
  border-radius: 4px;
  cursor: pointer;
}

.lang-btn.active {
  background-color: #4CAF50;
  color: white;
  border-color: #4CAF50;
}

.lang-content {
  display: none;
}

.lang-content.active {
  display: block;
}
</style>

<script>
function switchLanguage(lang) {
  // Hide all language content
  const contents = document.querySelectorAll('.lang-content');
  contents.forEach(content => {
    content.classList.remove('active');
  });
  
  // Show selected language content
  const selectedContent = document.getElementById(`content-${lang}`);
  if (selectedContent) {
    selectedContent.classList.add('active');
  }
  
  // Update button styles
  const buttons = document.querySelectorAll('.lang-btn');
  buttons.forEach(button => {
    button.classList.remove('active');
  });
  
  const selectedButton = document.getElementById(`lang-${lang}`);
  if (selectedButton) {
    selectedButton.classList.add('active');
  }
  
  // Save preference to localStorage
  localStorage.setItem('preferred_language', lang);
}

// Set initial language based on stored preference or default to English
document.addEventListener('DOMContentLoaded', function() {
  const preferredLanguage = localStorage.getItem('preferred_language') || 'en';
  switchLanguage(preferredLanguage);
});
</script>