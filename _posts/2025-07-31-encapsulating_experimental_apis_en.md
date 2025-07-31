---
title: 'Encapsulating Experimental APIs with Factory Pattern and Interfaces'
date: 2025-07-31
permalink: /posts/encapsulating_experimental_apis/en/
tags:
  - API Design
  - Factory Pattern
  - Kotlin
  - Best Practices
---

<div class="language-selector">
  <a href="/posts/encapsulating_experimental_apis/" class="lang-link">中文</a> | 
  <span class="lang-link active">English</span>
</div>

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

Here's a real example from my project showing how to encapsulate the experimental APIs from `kotlinx-datetime`:

#### 1. Define a Stable Interface

First, we define a clean interface that only includes what we really need:

```kotlin
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
     * Format: yyyy-MM-dd
     */
    fun formatDate(timestamp: Long): String

    /**
     * Get current system default timezone ID
     */
    fun getCurrentTimeZoneId(): String

    /**
     * Check if timestamp is expired
     * @param timestamp Expiration timestamp in milliseconds
     * @param bufferMillis Buffer time in milliseconds, default is 5000ms
     */
    fun isExpired(timestamp: Long, bufferMillis: Long = 5000): Boolean
}
```

This interface is clean and doesn't expose any experimental API details - it only focuses on our business needs.

#### 2. Implementation: Where All Experimental APIs Live

Next, create an implementation class where all experimental API usage is contained:

```kotlin
@file:OptIn(ExperimentalTime::class)
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
        return "${localDateTime.year}-${localDateTime.monthNumber.toString().padStart(2, '0')}-${localDateTime.dayOfMonth.toString().padStart(2, '0')} " +
                "${localDateTime.hour.toString().padStart(2, '0')}:${localDateTime.minute.toString().padStart(2, '0')}"
    }

    override fun formatDate(timestamp: Long): String {
        val instant = Instant.fromEpochMilliseconds(timestamp)
        val localDateTime = instant.toLocalDateTime(TimeZone.currentSystemDefault())
        return "${localDateTime.year}-${localDateTime.monthNumber.toString().padStart(2, '0')}-${localDateTime.dayOfMonth.toString().padStart(2, '0')}"
    }

    override fun getCurrentTimeZoneId(): String {
        return TimeZone.currentSystemDefault().id
    }

    override fun isExpired(timestamp: Long, bufferMillis: Long): Boolean {
        return Clock.System.now().toEpochMilliseconds() > (timestamp - bufferMillis)
    }
}
```

Notice that all experimental API usage is contained in this single file, with the `@OptIn` annotation only needed once at the file level.

#### 3. Factory: Providing Access to the Implementation

Finally, create a factory to manage instances:

```kotlin

/**
 * Time provider factory
 * Used to get TimeProvider instances
 */
object TimeProviderFactory {
    // Default instance, can be replaced for testing
    private var instance: TimeProvider = DefaultTimeProvider()

    /**
     * Get TimeProvider instance
     */
    fun getInstance(): TimeProvider {
        return instance
    }

    /**
     * Set custom TimeProvider instance
     * Mainly used for testing
     */
    fun setInstance(customInstance: TimeProvider) {
        instance = customInstance
    }
}
```

### Using the Pattern in Your Code

Now, instead of directly using experimental APIs throughout your codebase, you can use the factory:

```kotlin
class UserViewModel(
    private val repository: UserRepository,
    // Dependency can be injected, or obtained from factory
    private val timeProvider: TimeProvider = TimeProviderFactory.getInstance()
) : ViewModel() {
    
    fun formatLastLoginTime(timestamp: Long): String {
        return timeProvider.formatDateTime(timestamp)
    }
    
    fun checkSessionExpired(sessionExpiryTime: Long): Boolean {
        // No experimental API annotations needed here!
        return timeProvider.isExpired(sessionExpiryTime)
    }
}
```

The business code is now clean, with no experimental API annotations needed.

### Testing Made Easy

One of the biggest benefits of this pattern is how it simplifies testing. You can easily create a mock implementation:

```kotlin
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

<style>
.language-selector {
  margin-bottom: 20px;
  font-weight: bold;
}

.lang-link {
  text-decoration: none;
  color: #0366d6;
}

.lang-link.active {
  color: #24292e;
  pointer-events: none;
}

/* Code block styling */
.language-kotlin {
  background-color: #f6f8fa;
  border-radius: 6px;
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-size: 85%;
  line-height: 1.45;
  overflow: auto;
  padding: 16px;
}

.language-kotlin .keyword {
  color: #d73a49;
}

.language-kotlin .string {
  color: #032f62;
}

.language-kotlin .comment {
  color: #6a737d;
}

.language-kotlin .function {
  color: #6f42c1;
}
</style>
