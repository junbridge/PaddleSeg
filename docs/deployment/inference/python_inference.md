# Paddle Inference部署（Python）

## 1. 说明

本文档介绍使用Paddle Inference的Python接口在服务器端(Nvidia GPU或者X86 CPU)部署分割模型。

飞桨针对不同场景，提供了多个预测引擎部署模型（如下图），更多详细信息请参考[文档](https://paddleinference.paddlepaddle.org.cn/product_introduction/summary.html)。

![inference_ecosystem](https://user-images.githubusercontent.com/52520497/130720374-26947102-93ec-41e2-8207-38081dcc27aa.png)

## 2. 准备模型和数据

下载[样例模型](https://paddleseg.bj.bcebos.com/dygraph/demo/bisenet_demo_model.tar.gz)用于测试。如果要使用其他模型，大家可以使用[模型导出工具](../../model_export.md)。

```shell
wget https://paddleseg.bj.bcebos.com/dygraph/demo/bisenet_demo_model.tar.gz
tar zxvf bisenet_demo_model.tar.gz
```

下载cityscapes验证集中的一张[图片](https://paddleseg.bj.bcebos.com/dygraph/demo/cityscapes_demo.png)用于演示效果。如果大家的模型是使用其他数据集训练的，请自行准备测试图片。

```
wget https://paddleseg.bj.bcebos.com/dygraph/demo/cityscapes_demo.png
```

## 3. 准备部署环境

Paddle Inference是飞桨的原生推理库，提供服务端部署模型的功能。使用Paddle Inference的Python接口部署模型，只需要根据部署情况，安装PaddlePaddle。即是，Paddle Inference的Python接口集成在PaddlePaddle中。

在服务器端，Paddle Inference可以在Nvidia GPU或者X86 CPU上部署模型。Nvidia GPU部署模型计算速度快，X86 CPU部署模型应用范围广。

1) 准备X86 CPU部署环境

如果在X86 CPU上部署模型，请参考[文档](https://www.paddlepaddle.org.cn/install/quick?docurl=/documentation/docs/zh/install/pip/linux-pip.html)准备环境、安装CPU版本的PaddlePaddle（推荐版本>=2.1）。详细阅读安装文档底部描述，根据X86 CPU机器是否支持avx指令，选择安装正确版本的PaddlePaddle。

2) 准备Nvidia GPU部署环境

Paddle Inference在Nvidia GPU端部署模型，支持两种计算方式：Naive方式和TensorRT方式。TensorRT方式有多种计算精度，通常比Naive方式的计算速度更快。

如果在Nvidia GPU使用Naive方式部署模型，同样参考[文档](https://www.paddlepaddle.org.cn/install/quick?docurl=/documentation/docs/zh/install/pip/linux-pip.html)准备CUDA环境、安装GPU版本的PaddlePaddle（请详细阅读安装文档底部描述，推荐版本>=2.1）。比如：

```
# CUDA10.1的PaddlePaddle
python -m pip install paddlepaddle-gpu==2.1.2.post101 -f https://www.paddlepaddle.org.cn/whl/linux/mkl/avx/stable.html
```

如果在Nvidia GPU上使用TensorRT方式部署模型，同样参考[文档](https://www.paddlepaddle.org.cn/install/quick?docurl=/documentation/docs/zh/install/pip/linux-pip.html)准备CUDA环境（只支持CUDA10.1+cudnn7或者CUDA10.2+cudnn8.1）、安装对应GPU版本（支持TensorRT）的PaddlePaddle（请详细阅读安装文档底部描述，推荐版本>=2.1）。比如：

```
python -m pip install paddlepaddle-gpu==[版本号] -f https://www.paddlepaddle.org.cn/whl/stable/tensorrt.html
```

在Nvidia GPU上使用TensorRT方式部署模型，大家还需要下载TensorRT库。
CUDA10.1+cudnn7环境要求TensorRT 6.0，CUDA10.2+cudnn8.1环境要求TensorRT 7.1。
大家可以在[TensorRT官网](https://developer.nvidia.com/tensorrt)下载。这里只提供Ubuntu系统下TensorRT的下载链接。

```
wget https://paddle-inference-dist.bj.bcebos.com/tensorrt_test/cuda10.1-cudnn7.6-trt6.0.tar
wget https://paddle-inference-dist.bj.bcebos.com/tensorrt_test/cuda10.2-cudnn8.0-trt7.1.tgz
```

下载、解压TensorRT库，将TensorRT库的路径加入到LD_LIBRARY_PATH，`export LD_LIBRARY_PATH=/path/to/tensorrt/:${LD_LIBRARY_PATH}`

## 4. 预测

在PaddleSeg根目录，执行以下命令进行预测:

```shell
python deploy/python/infer.py \
    --config /path/to/deploy.yaml \
    --image_path /path/to/image/path/or/dir
```

**参数说明如下:**
|参数名|用途|是否必选项|默认值|
|-|-|-|-|
|config|**导出模型时生成的配置文件**, 而非configs目录下的配置文件|是|-|
|image_path|预测图片的路径或者目录或者文件列表|是|-|
|batch_size|单卡batch size|否|1|
|save_dir|保存预测结果的目录|否|output|
|device|预测执行设备，可选项有'cpu','gpu'|否|'gpu'|
|use_trt|是否开启TensorRT来加速预测|否|False|
|precision|启动TensorRT预测时的数值精度，可选项有'fp32','fp16','int8'|否|'fp32'|
|cpu_threads|使用cpu预测的线程数|否|10|
|enable_mkldnn|是否使用MKL-DNN加速cpu预测|否|False|
|benchmark|是否产出日志，包含环境、模型、配置、性能信息|否|False|
|with_argmax|对预测结果进行argmax操作|否|否|

**使用说明如下：**
* 如果在X86 CPU上部署模型，必须设置device为cpu，此外CPU部署的特有参数还有cpu_threads和enable_mkldnn。
* 如果在Nvidia GPU上使用Naive方式部署模型，必须设置device为gpu。
* 如果在Nvidia GPU上使用TensorRT方式部署模型，必须设置device为gpu、use_trt为True。这种方式支持三种数值精度：
    * 加载常规预测模型，设置precision为fp32，此时执行fp32数值精度
    * 加载常规预测模型，设置precision为fp16，此时执行fp16数值精度，可以加快推理速度
    * 加载量化预测模型，设置precision为int8，此时执行int8数值精度，可以加快推理速度
* 如果要开启`--benchmark`的话需要安装auto_log，请参考[安装方式](https://github.com/LDOUBLEV/AutoLog)。

测试样例的预测结果如下。

![cityscape_predict_demo.png](../../images/cityscapes_predict_demo.png)