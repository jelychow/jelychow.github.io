---
theme: fancy
excerpt: "Google IO 2025 展示了 Gemma 模型在移动端的强大能力，本文详细介绍如何使用 LiteRT 在 Android 设备上部署和运行 Gemma 模型，包括模型下载、CPU/GPU 加速对比及实际效果展示。"
---

前两天 Google IO 给我们演示 Gemma 模型在移动端的效果，说实话看着挺让人激动的，于是乎就体验了一番，说实话效果很棒，在这里就写一篇文章来给大家分享一下，如何在移动端 借助 LiteRT 来部署 Gemma 模型

## 1. 前置条件

1.  注册 huggingface 账号\
    [hugginguface](https://huggingface.co/) 是世界上最大的开源模型，数据集，机器学习的社区，上面会托管各式各样的模型，是每个 AI 爱好者的必备网站之一。

2.  科学上网\
    这个重要性不言而喻

## 2. 下载 app 或编译源码

### [下载地址此处可见](https://huggingface.co/litert-community/Gemma3-1B-IT)

### [源码地址此处可见](https://github.com/google-ai-edge/gallery/tree/main/Android)

## 3. 安装 App

## 4. 运行

打开之后会出现如下页面

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/41ced6ea374347db8435aec5e7c6268f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1748762610&x-orig-sign=NN32BuTa8nvccZJHapcIpKT29PE%3D)

## 5. 选择模块

这里以 AI chat model 为例

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/e8903efa6227435d99fe8d016f8e3e0a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1748762610&x-orig-sign=46qvuA9La3RH7pueEUuCq9tIsQQ%3D)

在这里我想选择第一个模型来体验，此处需要提醒使用者，一般来说模型越大占内存也会越大，使用起来手机发热，耗电量会增加，建议开发者在移动端谨慎选择，当然模型参数较小，量化都会降低精度。

选取了模型之后页面会跳转到浏览器，我这里使用的是 Chrome 浏览器。在之前我会在电脑上使用 chrome 注册 huggingface 账号，保存在 chrome 里面方便手机端同步账号。
跳转到 huggingface 获取认证之后会跳转回来开始下载流程，在下载完成之后页面上会出现这个页面

> 在这里有个好用的知识点推荐给大家，使用 Chrome Custom Tabs被用于提供一种轻量级的网页浏览体验，特别是在获取用户登录信息时。在有 chrome 浏览器的设备上他的体验会非常好，过度很自然，建议大家优先使用。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/914f53d7e8ed47769557cad0af3d4cdd~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1748762610&x-orig-sign=5OEwhSDJWw4NZ7aJeCn1kF9qPXM%3D)

## 6. 提问

然后开始 chat，举个例子：帮我简单叙述一下 kotlin 的 flow 和 stateflow 区别？

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/e61c05e2771445f1995ba5d9434bcd13~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1748762610&x-orig-sign=G4V%2FkOW45HjCYbpuGZdyB%2FnCCAg%3D)

然后这个模型会给你疯狂作答，下面会展示本次会话的状态。下面我们会注意到这个 model 是运行的 CPU 上的，我们可以指定 model 运行在 gpu 上看看效果。点击右上角的配置按钮，我们会得到如下的对话框

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/37c48c9ac4e6493285692940dc5c7355~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1748762610&x-orig-sign=JcEEUpHEMBJiO49kRGKE7NfXt6g%3D)
简单的设置一下加速器为 gpu 等待模型加载然后看看效果：

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/8733f8ae5cf644f580ac87c1b6b2934c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1748762610&x-orig-sign=wToKQf%2BPlsEwngHi610PUFqVESw%3D)

可以看到 GPU 的 token 生成速度是要远快于 CPU的，建议我们在体验的时候优先使用 GPU 来部署。

## 如果想看视频效果的[请见此链接](https://www.bilibili.com/video/BV1iWjuz1Evs/?vd_source=5700933c78eb2b70e4a486b309d6e808)

## 总结

祝周末愉快，码力十足🙂
