---
theme: smartblue
excerpt: "Compose 的甜甜圈理论堪称是 compose ui 理论基石，对于每一个学习 compose 的使用者来说是必须掌握的知识，下面我尝试从多个角度来学习"
---
# Jetpack Compose 经典学习之甜甜圈跳过

Compose 的[甜甜圈理论](https://www.jetpackcompose.app/articles/donut-hole-skipping-in-jetpack-compose#recomposition-scope)堪称是 compose ui 理论基石，对于每一个 compose 的使用者来说是必须掌握的知识。在此我也来从多个角度来分析 compose 的甜甜圈跳过。
## 第一步上代码。

```kt
@Preview
@Composable
fun MyComponent(modifier: Modifier = Modifier.padding(56.dp)) {
    var counter by remember { mutableStateOf(0) }
    LogCompositions("JetpackCompose.app", "MyComponent function")
    val readCounter = counter
    Box(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        contentAlignment = Alignment.Center
    ) {
        CustomButton(onClick = {
            counter++
        }) {
            LogCompositions("JetpackCompose.app", "CustomButton scope")
            CustomText(
                text = "Counter: $counter",
                modifier = Modifier
                    .clickable {
                        counter++
                    },
            )
        }
    }
}

@Composable
fun CustomText(
    text: String,
    modifier: Modifier = Modifier.padding(top = 46.dp),
) {
    LogCompositions("JetpackCompose.app", "CustomText function")
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

@Composable
fun CustomButton(
    onClick: () -> Unit,
    content: @Composable () -> Unit
) {
    LogCompositions("JetpackCompose.app", "CustomButton function")
    Button(onClick = onClick, modifier = Modifier.padding(16.dp)) {
        LogCompositions("JetpackCompose.app", "Button function")
        content()
    }
}
class Ref(var value: Int)

@Composable
inline fun LogCompositions(tag: String, msg: String) {
    val ref = remember { Ref(0) }
    SideEffect { ref.value++ }
    Log.d(tag, "Compositions: $msg ${ref.value}")
}
```
## 留一秒钟猜一猜 log 是什么

```logs
Compositions: MyComponent function 0
Compositions: CustomButton function 0
Compositions: Button function 0
Compositions: CustomButton scope 0
Compositions: CustomText function 0
```
## 点击一下看看？
可以看到第一次重组的时候，所有 composable 的 function 都被调用了。这个时候如果我点击了按钮，想想会发生什么？
```logs
Compositions: MyComponent function 1
Compositions: CustomButton scope 1
Compositions: CustomText function 1
```
可以看到只有 MyComponent function-> CustomButton scope-> CustomText function, 这时候 CustomButton func 并没有参与重组,是不是很奇怪？这就是 compose 独特，神奇的一面，他不像以往的框架，会最小化重绘最小树，compose 会智能的跳过不需要重组的方法与范围。那么 compose 是如何知道哪个地方需要重组，那个地方不需要重组呢，我们暂且放下，先看下一个 case。

## 改一下看看？
```kt
//    val readCounter = counter
```
然后我们正常点击一下看看log
```logs
... 同上
// 点击一下
Compositions: CustomButton scope 1
Compositions: CustomText function 1
```
可以看到 MyComponent 并没有发生重组，这之间重组的秘密到底是什么？具体原理我总结在这篇[文章](https://jelychow.github.io/Jetpack-Compose-%E7%8A%B6%E6%80%81%E4%BE%9D%E8%B5%96%E8%BF%BD%E8%B8%AA%E5%8E%9F%E7%90%86%E8%AF%A6%E8%A7%A3/)里面了。

## 再改一点试试看
```kt 
text = "text",
```
我们把 text 改一下看看结果如何
```logs
... 同上

Compositions: CustomButton scope 1
Compositions: CustomText function 1
```
奇怪为什么 CustomText 为什么会发生重组？让我们看看到底是什么原因？现在我们回头看下 CustomText 的调用处。
```kt
    CustomText(
        text = "Counter: $counter",
        modifier = Modifier
            .clickable {
                counter++
            },
    )
```
我们看到有 2 个地方分别使用了 counter，那到底是哪个地方引起的呢，我们分别注释调看看，
### 注释掉 text = "Counter",
这个时候我们会发现怎么点，所有重组都没了，这是为什么呢？
```logs
...
```
想想为什么？  
许久沉思之后我们想到上面提到了，如果能捕获重组必须得建立 read 依赖，才能够触发重组，如果注释 `counter` 使用，目前没有一个 read 依赖，所以也就不会触发重组了。
### 注释掉另一个呢，
```kt
.clickable {
    counter++
 }
```

```logs
... 同上
Compositions: CustomButton scope 1
```
显示  scope 里面被使用，但是方法没有被调用，为什么呢，
## 最后一个问题 composable lambda 和普通 lambda 到底有何不同？
## 