---
highlight: vs2015
theme: smartblue
excerpt: "为什么选择Jetpack Compose？全面解析Android新一代UI开发框架的优势，从声明式UI到性能优化，带你深入理解现代Android UI开发。"
---
### 前言
本系列主要目的是分享对于 Now In Android 系列的个人理解，会尝试从自己的角度来分析 Android 现况，以及架构发展等诸多方面。

### 现代应用架构

现代应用架构鼓励采用以下方法及其他一些方法：
- A reactive and layered architecture.
- Unidirectional Data Flow (UDF) in all layers of the app.
- A UI layer with state holders to manage the complexity of the UI.
- Coroutines and flows.
- Dependency injection best practices.

从上面的架构上我们可以看到 Android 官方实际上正在鼓励我们采用一些新的技术来进行编程。但是随之而来的第一句话就会让大多数人困惑。

> A reactive and layered architecture.

一个响应式的和分层的架构，分层我们大家都有一点的了解：UI->Domain->Data。但是说到响应式（Reactive），究竟什么是响应式大家还是一头雾水。下面来看一段我从 VUE 官方文档上摘选的一段话：


>  **什么是响应性?**  
> 这个术语在今天的各种编程讨论中经常出现，但人们说它的时候究竟是想表达什么意思呢？本质上，响应性是一种可以使我们声明式地处理变化的编程范式

作为 Android 开发，想必对响应式编程多少有一些了解，例如 RxJava，Kotlin Flow，他们都是响应式编程的框架。下面看一段 Kotlin flow 代码：


```Kotlin
fun main() = runBlocking {
    // 创建一个 Flow
    val flow = flow {
        for (i in 1..5) {
            delay(1000) // 模拟延迟
            emit(i) // 发射数据
        }
    }

    // 订阅 Flow
    flow.collect { value ->
        println("Received: $value")
    }
}
```
kotlin flow 拥有众多非常有用的操作符，本质上这些操作符都是声明式的。那么在我们说了这么多之后这时候该唤出我们的主角了：Compose UI。
### Compose UI
Jetpack Compose 是一个适用于 Android 的新式声明性界面工具包。Compose 提供声明性 API，让您可在不以命令方式改变前端视图的情况下呈现应用界面，从而使编写和维护应用界面变得更加容易。
#### 声明式 UI

```kt
@Composable
fun Greeting(name: String) {
    Text(text = "Hello, $name!")
}
```

#### 命令式 UI


```kt
val textView: TextView = findViewById(R.id.textView)
textView.text = "Hello, $name!"
```
从上面的两段代码，我们可以看到命令式与声明式的几点区别：
- 命令式 UI 重点关心如何做，声明式 UI 关心他是什么
- 命令式 UI 手动更新状态，声明式 UI 自动更新状态
- 可服用性 声明式 UI 较高

上面的代码片段较少，关于比较两者之间的可维护性，大家可能没什么概念。假设一下，有一个非常复杂的页面，有二十多个组件要展示在一个页面里面，这时候你需要维护 xml，viewmodel，以及 fragment 里面的 UI 想想就刺激，刚开始开发的时候还好吧，但是后续维护的时候，你总是会这里漏一点，那里漏一点，测试也不方便，非常容易产生 bug。  
在这个情况下 Android 的演进方向应当是声明式的，记得有段时间 Android 曾经推出过在 xml 里面来写 state 数据，但是 xml与 kotlin 代码 又不能一起写，维护起来也不是很方便，慢慢的也被人遗忘。  
有一说一 Android 对于架构的演进明显是受到了整个行业的影响的，但他的步调确实最慢的。2013 年 React 出现，2014 年 VUE 出现，2019 年 Swift UI 出现，2020 compose 才出现。说到底 Compose UI 的出现，其实是被动的，在与行业发展与技术债之间被迫做出的努力。这一点在 Compose 官方文档也有说明：
> 在过去的几年中，整个行业已开始转向声明性界面模型，该模型大大简化了与构建和更新界面关联的工程任务。该技术的工作原理是在概念上从头开始重新生成整个屏幕，然后仅执行必要的更改。此方法可避免手动更新有状态视图层次结构的复杂性。Compose 是一个声明性界面框架。

结尾：  
Compose UI 的诞生一部分原因是为了满足现代架构的需要，也有一部分是看到了行业发展被迫产生的应对措施。我建议大家如果有时间的话看一下 React 与 Vue.js 的文档。你会发现 Compose 的设计思想，包括很多概念不仅与他们相像，简直就是一模一样 🤣。但是话又说回来 Compose UI 可能会在某些场景不如以往的 xml，但是他必定是以后的发展方向，在这里我建议每个 Android 程序员都必须掌握，可以这么说如果你掌握了 compose ui 的编程思想，那么以后 React，Vue，鸿蒙都可以尝试，这样岂不美哉?