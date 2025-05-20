---
title: "ViewModel内存泄漏问题解析"
collection: publications
category: manuscripts
permalink: /publication/2025-05-19-android-questions
excerpt: "深入分析Android ViewModel持有Context导致的内存泄漏问题及解决方案"
date: 2025-05-19
venue: 'Android开发技术'
theme: awesome-green
---

ViewModel持有Context会导致内存泄漏主要是因为**生命周期不匹配**的问题。这种情况非常常见，但却容易被忽视。

## 内存泄漏原理

### 生命周期不一致

- **ViewModel生命周期**: 比Activity长，在配置更改（如屏幕旋转）时仍然存活
- **Activity Context生命周期**: 与Activity绑定，Activity重建时会销毁

当ViewModel持有Activity的Context时：
1. 当配置更改发生时，Activity被销毁并重建
2. ViewModel保持存活，并持有对旧Activity的引用
3. 旧Activity无法被垃圾回收，造成内存泄漏

### 引用链问题

Activity Context包含对整个视图层次结构的引用，这意味着ViewModel间接持有了大量对象：
- UI组件
- Drawable资源
- 布局对象
- 适配器和数据集

随着配置变更次数增加，泄漏的 Activity 实例会累积，最终可能导致OOM崩溃。

## 解决方案

### 1. 使用Application Context

```kotlin
class MyViewModel(application: Application) : AndroidViewModel(application) {
    private val appContext = application.applicationContext
    
    fun doSomething() {
        // 使用appContext而非Activity context
    }
}
```

### 2. 使用依赖注入管理生命周期

通过 Hilt 或 Koin 等框架配合 lifecycle-aware components 管理依赖，避免手动持有 Context。

## 最佳实践

1. 原则上ViewModel不应该持有任何Context引用
2. 如必须使用Context，使用AndroidViewModel并仅使用Application Context
3. 将需要Context的操作移至Repository层或UseCase
4. 使用架构组件如LiveData/Flow进行通信而非直接引用

遵循这些原则可以有效避免因ViewModel持有Context导致的内存泄漏问题。

## 为什么依赖注入可以避免内存泄漏？
举例 例如 koin:  
- 总是使用 androidContext() 提供应用级 Context
- 正确设置组件作用域，避免长生命周期持有短生命周期组件
- 使用 by viewModel() 而非手动创建 ViewModel 实例
- 注意关闭自定义作用域 scope.close()