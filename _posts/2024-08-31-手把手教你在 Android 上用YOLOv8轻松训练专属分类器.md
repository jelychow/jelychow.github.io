---
theme: fancy
excerpt: "最近在了解机器学习方面的知识，了解了一些分类相关的算法实现，于是乎也想上手玩玩，毕竟实践出真知。看了看网上有很多 pre-trained 模型，可以实现很多强大的功能，不过还是想基于这些模型做一个二次训练做一个自己的模型，这样才更有成就感，自己也能在实践中学习到一些书本上没有的细节。
"

---

## 前言

最近在了解机器学习方面的知识，了解了一些分类相关的算法实现，于是乎也想上手玩玩，毕竟实践出真知。看了看网上有很多 pre-trained 模型，可以实现很多强大的功能，不过还是想基于这些模型做一个二次训练做一个自己的模型，这样才更有成就感，自己也能在实践中学习到一些书本上没有的细节。

## 灵感

现实中很多物体可以做分类，不过我的灵感来自我的2只可爱的狗狗，一只是金毛，一只是只串串。我一直好奇串串是个什么品种，所以我选择自己做一个狗狗分类器，这样我能比较直观的感受到自己训练的成果是不是准确，毕竟如果训练了一个自己不懂的物体，无法直接的判断模型的好坏。

## 框架选择

YOLOv8 是一个在目标检测，分割领域使用的最多的框架，使用他主要是他封装的比较齐全，使用简单，毕竟作为一个 Android 程序员我的主要领域都集成在了 Android，Kotlin 这块。

### 数据集准备

分类算法的数据集基本上都类似于下面的格式

#### 数据集格式介绍

```js
cifar-10-/
|
|-- train/
|   |-- airplane/
|   |   |-- 10008_airplane.png
|   |   |-- 10009_airplane.png
|   |   |-- ...
|   |
|   |-- automobile/
|   |   |-- 1000_automobile.png
|   |   |-- 1001_automobile.png
|   |   |-- ...
|   |
|   |-- bird/
|   |   |-- 10014_bird.png
|   |   |-- 10015_bird.png
|   |   |-- ...
|   |
|   |-- ...
|
|-- test/
|   |-- airplane/
|   |   |-- 10_airplane.png
|   |   |-- 11_airplane.png
|   |   |-- ...
|   |
|   |-- automobile/
|   |   |-- 100_automobile.png
|   |   |-- 101_automobile.png
|   |   |-- ...
|   |
|   |-- bird/
|   |   |-- 1000_bird.png
|   |   |-- 1001_bird.png
|   |   |-- ...
|   |
|   |-- ...
|
|-- val/ (optional)
|   |-- airplane/
|   |   |-- 105_airplane.png
|   |   |-- 106_airplane.png
|   |   |-- ...
|   |
|   |-- automobile/
|   |   |-- 102_automobile.png
|   |   |-- 103_automobile.png
|   |   |-- ...
|   |
|   |-- bird/
|   |   |-- 1045_bird.png
|   |   |-- 1046_bird.png
|   |   |-- ...
|   |
|   |-- ...
```

> 数据集必须在根目录下以特定的分割目录结构进行组织，以方便正确的训练、测试和可选的验证过程。这种结构包括用于训练（train）和测试（test）阶段的单独目录，以及用于验证（val）的可选目录。

> 每个目录都应为数据集中的每个类包含一个子目录。子目录以相应的类命名，并包含该类的所有图像。确保每个图像文件都有唯一的名称，并以 JPEG 或 PNG 等通用格式存储。对于 Ultralytics YOLO 分类任务，数据集必须在根目录下以特定的分割目录结构进行组织，以方便正确的训练、测试和可选的验证过程。这种结构包括用于训练（train）和测试（test）阶段的单独目录，以及用于验证（val）的可选目录。

> 每个目录都应为数据集中的每个类包含一个子目录。子目录以相应的类命名，并包含该类的所有图像。确保每个图像文件都有唯一的名称，并以 JPEG 或 PNG 等通用格式存储。

一般来说上述的分类数据集是比较通用的，不仅仅适用于 YOLOv8 的分类数据集，也适用于 mobilenet，resnet 等分类框架。

#### 数据集获取

深度学习需要海量的数据集来训练，以保住模型的能力，我们可以自己准备数据集，或者使用网上的开源数据集，一般来说对于开发者来说，有两个地方可以免费获得 dataset（数据集），并且轻易的使用。一个是 <https://huggingface.co/，> 另一个是 <https://www.kaggle.com/> ，本文中以 huggingface 为例。打开 hugging face，选择 dataset 可以看到下面的页面。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/04ce6a90173649e9a2095d588ca5117f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1741349237&x-orig-sign=1OOOekyAAGtuNs7RjE%2FWJe6dWdo%3D)

搜索框里面我们输入 dog 选择我们需要的数据集，note：请选择 train 数据集 和 test 数据集都存在的dataset，否则将会存在以下风险，因为模型的训练是需要测试数据进行验证回馈，调整损失函数进行下一步训练的。

1.  **无法评估泛化能力**：你不能验证模型在未见数据上的表现。
2.  **过拟合风险**：可能无法检测到模型是否过拟合。

##### 数据集结构展示

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/c48116ef5e034342b4af32ae83305214~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1741349237&x-orig-sign=jMCA34m06zVFxuFCJ22eC7pUb2o%3D)
以上面的这个数据集为例，一般数据集结构可以使用 viewer 进行查看，通过上面的 split 可以看到当前数据集的完整性，是否存在 train，test，val split。通过切换我们可以知道这个dataset 的 test split 并没有完整分类，是无法直接用于分类任务的，我们需要换一个数据集来实现。

##### 数据集使用

一般来说网站上都会提供完整的数据集的使用方法，示例如下
![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/0c70c6d8c2544d5692360dfffefbe3b2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1741349237&x-orig-sign=43m%2BwGEzpL5dcWSr9Abhj%2F4Uu%2Bk%3D)
如果我们点击每个按钮就会有对应的集成代码，本例中以 datasets 库为例，点击之后会出现一个提示框出现类似下面的代码：

```Python
from datasets import load_dataset

ds = load_dataset("amaye15/stanford-dogs")
```

如果在 pycharm 上直接运行，你将会下载上面的数据集。放心数据集只会运行一次。现在你有了完整的数据集了，你需要的就是按照上面的数据集的结构将图片copy一份过来。

### 训练/ Train

现在万事俱备，只需要我们按照官方文档 copy 出来代码

```Python
from ultralytics import YOLO
# Load a model
model = YOLO("yolov8n-cls.pt")  # load a pretrained model (recommended for training)
# Train the model
results = model.train(data="path/to/dataset", epochs=100, imgsz=640)
```

点击开始，训练就正式开始了。
如果不出意外的话，意外就要开始了，你会发现你的进度条不动了，这时候你可能察觉不到异常，因为 YOLOv8 默认的训练是使用 cuda 的，是需要 NVIDIA 显卡来支持的，这时候你需要指定使用 cpu 来训练，当然如果你是高贵的 **mac pro** 用户，你可以指定设备为 mps 来训练，训练能力排序为 cuda>mps>cpu。

```python
# Ensure the model uses CPU
model.to('cpu')
```

修复问题我们接着训练，当然为了节省我们不需要讲训练次数 epoch 设置成 100，可以设置成 30 或者50 先看看效果。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/d0a4c37f22b74331b36e4eeb3a23ec09~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1741349237&x-orig-sign=gLgGjk7SR1IH3oK5O9u9s3%2FwYOY%3D)
如果一切顺利的话，控制台会实时展示训练的成果。

### 模型验证与导出

#### 模型预测

```Python
from ultralytics import YOLO
# Load a model
checkpoint = 'your_model_path.pt'
model = YOLO(checkpoint)  # pretrained YOLOv8n model
# Run batched inference on a list of images
results = model(["im1.jpg", "im2.jpg"])  # return a list of Results objects
# Process results list
for result in results:
    boxes = result.boxes  # Boxes object for bounding box outputs
    masks = result.masks  # Masks object for segmentation masks outputs
    keypoints = result.keypoints  # Keypoints object for pose outputs
    probs = result.probs  # Probs object for classification outputs
    obb = result.obb  # Oriented boxes object for OBB outputs
    result.show()  # display to screen
    result.save(filename="result.jpg")  # save to disk
```

我们可以使用上面的代码找几张测试图片预测一下当前模型，体验一下自己的劳动成果。checkpoint 这个参数可以在控制台看到，一般路径为：runs/classify/trainX/weights。

#### 模型导出

YOLOv8 的模型导出非常简单，可以说是傻瓜式。

```Python
from ultralytics import YOLO
# Load a model
model = YOLO("path/to/best.pt")  # 加载你自己训练数据集
# 导出为你需要的数据集
model.export(format="onnx")
```

一般来说移动端的部署框架选择很多，例如 腾讯的 ncnn，Facebook的 pytorch ，Google 的 tensorflow-lite，百度的 paddle，微软的 onnx-runtime。在这里我选择的是 onnx runtime 框架进行部署。确定了部署的框架之后我们就需要导出模型文件了，这里我的设置如上面所示。

### 在Android 手机上跑起来

我们可以基于 onnx runtime 原有 [demo](https://github.com/microsoft/onnxruntime-inference-examples/tree/main/mobile/examples/image_classification/android) 稍微做下改造，将里面模型路径换成我们自己的路径，里面的 classes txt 换成我们自己的模型 labels 这样就可以成功部署啦。
部署之后我们会兴冲冲打开自己App 验证一下模型，可是我们可能会发现自己的模型并不能正确识别自己的狗狗。在这里我就不绕弯子了，由于 onnx 输入图片已经做了 归一化 如果不了解的可以看一下相关介绍，在这里我们只需要在 上面的训练代码添加一个参数即可，

```python
model.train(
    data='dataset',  # Path to your dataset configuration file
    epochs=5,  # Number of epochs
    workers=4,  # Number of workers
    batch_size=16,  # Batch size
    augmented=True,  # Use augmented data
)

model.export(format='onnx',simplify=True)
```

#### 重新验证

经过重新部署验证之后，用自己的狗狗 看看自己训练的结果，终于知道自己的狗宝像哪个品种了了，原来是博美呀

![f1d667223a561637f6bd66b91ebc40d.jpg](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/9703de71bae04dc781881e6f91e2f8eb~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1741349237&x-orig-sign=xvq8bX7da61xGe5V2f3KIg6W3v0%3D)

![6fb85081a12e08ae36873824d36d4ee.jpg](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/3ef60c6d71bb48a7ba635911b2ec1411~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1741349237&x-orig-sign=9Obmo1FNAGngx4Davgc5gyvxAkU%3D)

## 结语

好啦，上面就是我的分享啦，祝周末愉快。附上自己的 [repo](https://github.com/jelychow/Android_Yolo8_Dog_Classfication)
