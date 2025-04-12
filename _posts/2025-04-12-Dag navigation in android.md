---
theme: awesome-green
excerpt: "excerpt: "Dag navigation in android""

---
### 使用 DAG （有向无环图）管理复杂依赖

#### 前言
在 Android 项目中，DAG（有向无环图）被用于管理应用的导航流程，管理项目依赖。我们常见的很多工具例如 gradle 里面也会应用 DAG 来检测循环依赖。DAG 的核心思想是通过节点和边的有向连接来表示任务或步骤之间的依赖关系，确保任务按照正确的顺序执行，同时避免循环依赖。本文将以一个简单的例子来介绍使用 DAG 来实现项目里面的复杂导航。

#### 需求介绍
打开 app 的时候，如果没有授权过都会弹出一个**隐私协议**，如果没有登录需要进入**登录页面**，获取登录态才可以进入**首页**，这是一个非常初级的需求，相信大部分人用 **if else** 都可以完成，但是如果一个 app 有很多检查项，很多流程那么使用  **if else** 怕是很难满足业务需求。

#### 需求设计

#### 1. **DAG的核心组件**
- **DagNode**: 代表DAG中的一个节点，包含节点的唯一标识符（`id`）、数据（`data`）、依赖节点（`dependencies`）以及条件检查函数（`conditionCheck`）。节点可以依赖于其他节点，并且可以设置条件来决定节点是否可以被执行。
- **DagManager**: 负责管理所有的DagNode，提供节点的添加、删除、查询、拓扑排序等功能。它还负责检测循环依赖，并确保DAG的正确性。
- **FlowManager**: 使用DagManager来管理应用的导航流程。它根据用户偏好（如是否同意隐私政策、是否登录等）来决定当前应该显示的界面。
> 上面的架构设计好了，然后动动手指头让 AI 生成一下。

#### 2. 代码片段

Node 节点片段

```kt
/**
 * DAG节点，代表流程中的一个步骤或任务
 */
class DagNode<T>(
    val id: String,
    val data: T,
    val dependencies: MutableSet<DagNode<T>> = mutableSetOf(),
    private var conditionCheck: (() -> Boolean)? = null,
) {
    // 添加依赖节点
    fun dependsOn(node: DagNode<T>) {
        dependencies.add(node)
    }
    
    // 添加多个依赖节点
    fun dependsOn(vararg nodes: DagNode<T>) {
        nodes.forEach { dependencies.add(it) }
    }
    
    // 检查是否依赖于指定节点
    fun isDependentOn(node: DagNode<T>): Boolean {
        if (dependencies.contains(node)) return true
        
        // 递归检查间接依赖
        for (dependency in dependencies) {
            if (dependency.isDependentOn(node)) return true
        }
        
        return false
    }
    
    ...
}
```

DagManager

```kt
/**
 * DAG管理器，负责管理节点和执行流程
 */
class DagManager<T> {
    private val nodes = mutableMapOf<String, DagNode<T>>()

    // 添加节点
    fun addNode(node: DagNode<T>) {
        // 检查是否会形成循环依赖
        for (existingNode in nodes.values) {
            require(!(node.isDependentOn(existingNode) && existingNode.isDependentOn(node))) {
                "添加节点 ${node.id} 会导致循环依赖"
            }
        }
        nodes[node.id] = node
    }
}
```

FlowManager

```kt
/**
 * 应用流程管理器，使用DAG管理应用的导航流程
 */
class FlowManager(private val userPreferences: UserPreferences) {
    

    
    // 创建DAG管理器
    private val dagManager = DagManager<FlowStep>()
    
    init {
        // 创建流程节点
        val privacyNode = DagNode("privacy", FlowStep.PRIVACY_POLICY)
        val loginNode = DagNode("login", FlowStep.LOGIN)
        val mainNode = DagNode("main", FlowStep.MAIN)
        
        // 建立依赖关系
        loginNode.dependsOn(privacyNode)  // 登录依赖于隐私政策同意
        mainNode.dependsOn(loginNode)     // 主界面依赖于登录
        
        // 添加节点到管理器
        dagManager.addNode(privacyNode)
        dagManager.addNode(loginNode)
        dagManager.addNode(mainNode)
    }
    ...
}
```

#### 3. **创建DAG**

##### 3.1 创建DagNode
首先，你需要创建DagNode来表示流程中的每个步骤。每个节点可以设置条件检查函数，来决定该节点是否可以被执行。

```kotlin
// 创建流程节点
 val privacyNode = DagNode("privacy", Screen.PrivacyPolicy.route)
 val loginNode = DagNode("login", Screen.Login.route)
 val mainNode = DagNode("main", Screen.Main.route)
 // 建立依赖关系
 loginNode.dependsOn(privacyNode)  // 登录依赖于隐私政策同意
 mainNode.dependsOn(loginNode)     // 主界面依赖于登录       
```
上面的代码看起来有些偏向于比较传统的命令式编程方式，我们能不能使用 DSL 声明式语法来实现，让语义更简单明了

```kt
val privacyNode = DagNode("privacy", Screen.PrivacyPolicy.route) {
            condition { userPreferences.hasAgreedToPrivacyPolicy() }
        }

val loginNode = DagNode("login", Screen.Login.route) {
            +privacyNode  // 登录依赖于隐私政策同意
            condition { userPreferences.isLoggedIn() }
        }
```
1. 使用高阶函数作为构造器参数：
    -  添加了 initBlock: (DagNode<T>.() -> Unit)? 作为构造函数的可选参数
    - 这是一个接收者函数类型，允许在其作用域内直接访问 DagNode，最重要的是他作为最后一个参数可以进行 lamada 优化
2. 实现运算符重载：
    - 添加了 operator fun DagNode<T>.unaryPlus() 以支持 +node 语法
    - 这使依赖关系的表达更加简洁明了  
      **note**：unaryPlus 操作符很好记，一元操作符，意思只有一个 + 对象，而 plus 是二元的
3. 添加 DSL 专用方法：
    - 创建 condition { ... } 方法，将条件检查逻辑与节点创建紧密集成

##### 3.2 创建DagManager
接下来，你需要创建DagManager来管理这些节点。使用我们上一节学到的方法来简单的改造一下方法，实现如下调用

```kotlin
val dagManager = DagManager<String>()
dagManager.dag {
    +privacyNode
    +loginNode
    +mainNode
}
```

##### 3.3 获取当前步骤
通过DagManager的`getStartNode`方法，获取当前应该执行的步骤。本算法是基于**入度** 来实现，从依赖为 0 的节点开始遍历，然后解除依赖，如果没有解除过的同学可以了练练[课程表](https://leetcode.cn/problems/course-schedule/description/)这个题目，里面有更详细的解释。

```kotlin
val currentStep = dagManager.getStartNode()?.data ?: Main.route
```


#### 4. **DAG的优势**
- **清晰的依赖关系**: DAG通过节点和边的连接清晰地表示了任务之间的依赖关系，使得复杂的流程变得易于管理。
- **避免循环依赖**: DagManager会自动检测循环依赖，并抛出异常，确保流程的正确性。
- **条件检查**: 每个节点可以设置条件检查函数，确保只有在满足特定条件时才会执行该节点。

#### 5. **总结**
DAG在项目中主要用于管理应用的导航流程，确保用户按照正确的顺序完成某些任务。通过DagNode和DagManager，开发者可以轻松地定义和管理复杂的任务依赖关系，并确保流程的正确性。DAG的使用不仅提高了代码的可读性和可维护性，还增强了应用的健壮性。

通过本文的介绍，你应该能够理解如何在项目中使用DAG来管理任务流程，并能够根据实际需求进行扩展和优化。
最后附上[项目传送门](https://github.com/jelychow/DAG)