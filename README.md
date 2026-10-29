# YOLOv8 Pose：SOPHON BM1684 / BM1684X 姿态与跌倒检测


面向 SOPHON TPU 的 C++ / BMCV 例程，使用 YOLOv8 Pose 提取人体关键点，并在视频后处理中实现简单的跌倒判定。适合需要阅读模型转换、PCIe/SoC 构建与视频推理流程的开发者。

YOLOv8 pose estimation and a simple fall-detection demo for SOPHON BM1684/BM1684X, with BMCV preprocessing and C++ inference.

## 从哪里开始

- [构建入口](cpp/yolov8_bmcv/CMakeLists.txt)：`TARGET_ARCH=pcie`（默认）或 `soc`；SoC 构建需要 `SDK` 路径和 aarch64 工具链。
- [程序参数与输入分支](cpp/yolov8_bmcv/main.cpp)：`--input`、`--bmodel`、`--dev_id`、`--conf_thresh`、`--nms_thresh`、`--classnames`。
- [推理与姿态后处理](cpp/yolov8_bmcv/yolov8.cpp)：模型执行、关键点绘制和跌倒相关逻辑。
- `models/` 保存现有 ONNX / BModel；`datasets/` 保存现有视频与类别文件；`pics/` 为原有流程截图。

本次文档整理仅静态核对源码、目录与命令，没有连接 TPU、重新编译模型或测量帧率。下方截图和既有结果是仓库原有资料，不是本轮重测。

## 已知使用边界

- 需要匹配的 libsophon、sophon-opencv、sophon-ffmpeg 和硬件环境，普通 CPU/macOS 环境不能直接替代。
- 请选择与芯片和运行时匹配的 BModel。当前默认代码中的 `models/BM1684X/...` 路径与仓库目录不一致，应显式传入实际存在的模型。
- 当前仓库没有 `datasets/test` 目录。图片输入需换成你准备的图片目录；视频可使用已有 `datasets/dance_1080P.mp4` 等文件。
- `main.cpp` 的现有输入检查可能提前拒绝 RTSP 地址；该路径仍需处理/实机验证，不应仅凭示例宣称可用。
- 跌倒判定是演示性后处理，未提供通用准确率、召回率或安全场景保证。当前仓库没有根目录 LICENSE，使用代码、模型及第三方组件前请核对各自授权。

## 目录
* [1. 简介](#简介)
* [2. 特性](#环境准备)
* [3. 准备模型与数据](#模型转换和量化)
* [4. 模型编译](#程序编译)
* [5. 例程测试](#推理测试)
  

## 简介

本例程是基于国产比特大陆AI芯片BM1684和BM1684X完成YOLOv8肢体关节点检测提取的算法移植，同时后处理实现简单的实时视频的跌倒检测任务。

其中，模型文件存放在models文件夹下；测试视频与类别文件存放在datasets文件夹下；程序代码存放在/cpp/yolov8_bmcv/下。


## 环境准备

### x86/arm PCIe平台
如果您在x86/arm平台安装了PCIe加速卡（如SC系列加速卡），可以直接使用它作为开发环境和运行环境。您需要安装libsophon、sophon-opencv和sophon-ffmpeg，

### SoC平台
如果您使用SoC平台（如SE、SM系列边缘设备），刷机后在`/opt/sophon/`下已经预装了相应的libsophon、sophon-opencv和sophon-ffmpeg运行库包，可直接使用它作为运行环境。通常还需要一台x86主机作为开发环境，用于交叉编译C++程序。

### 交叉编译环境搭建
需要在x86主机上使用SOPHON SDK搭建交叉编译环境，将程序所依赖的头文件和库文件打包至soc-sdk目录中。
1. 安装交叉编译工具链
    ```bash
    sudo apt-get install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu
    ```
    如果报错：`/lib/aarch64-linux-gnu/libc.so.6: version 'GLIBC_2.33' not found`。
    这是由于您主机上的交叉编译工具链版本太高导致，可以在[linaro官方网站](https://releases.linaro.org/components/toolchain/binaries/7.5-2019.12/aarch64-linux-gnu/)下载不高于边缘设备gcc版本的交叉编译工具链。

    这里提供一个ubuntu配置的例子：
    ```bash
    sudo apt remove cpp-*-aarch64-linux-gnu

    wget -nd https://releases.linaro.org/components/toolchain/binaries/7.5-2019.12/aarch64-linux-gnu/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu.tar.xz
    
    tar xvf gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu.tar.xz
    
    export PATH=$PWD/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu/bin/:\$PATH >> ~/.bashrc
    
    source ~/.bashrc
    ```

2. 打包libsophon

    从[算能官网](https://developer.sophgo.com/site/index/material/28/all.html)上下载符合[环境准备](#环境准备)的sophon-img安装包，其中包括libsophon_soc_x.y.z_aarch64.tar.gz，x.y.z表示版本号，并进行解压。

    ```bash
    # 创建依赖文件的根目录
    mkdir -p soc-sdk
    # 解压libsophon_soc_x.y.z_aarch64.tar.gz
    tar -zxf libsophon_soc_${x.y.z}_aarch64.tar.gz
    # 将相关的库目录和头文件目录拷贝到依赖文件根目录下
    cp -rf libsophon_soc_${x.y.z}_aarch64/opt/sophon/libsophon-${x.y.z}/lib ${soc-sdk}
    cp -rf libsophon_soc_${x.y.z}_aarch64/opt/sophon/libsophon-${x.y.z}/include ${soc-sdk}
    ```

3. 打包sophon-ffmpeg和sophon-opencv

    从[算能官网](https://developer.sophgo.com/site/index/material/28/all.html)上下载符合[环境准备](#环境准备)的sophon-mw安装包，其中包括sophon-mw-soc_x.y.z_aarch64.tar.gz，x.y.z表示版本号，并进行解压。
    ```bash
    # 解压sophon-mw-soc_x.y.z_aarch64.tar.gz
    tar -zxf sophon-mw-soc_${x.y.z}_aarch64.tar.gz
    # 将ffmpeg和opencv的库目录和头文件目录拷贝到soc-sdk目录下
    cp -rf sophon-mw-soc_${x.y.z}_aarch64/opt/sophon/sophon-ffmpeg_${x.y.z}/lib ${soc-sdk}
    cp -rf sophon-mw-soc_${x.y.z}_aarch64/opt/sophon/sophon-ffmpeg_${x.y.z}/include ${soc-sdk}
    cp -rf sophon-mw-soc_${x.y.z}_aarch64/opt/sophon/sophon-opencv_${x.y.z}/lib ${soc-sdk}
    cp -rf sophon-mw-soc_${x.y.z}_aarch64/opt/sophon/sophon-opencv_${x.y.z}/include ${soc-sdk}
    ```

这里，交叉编译环境已经搭建完成，接下来可以使用打包好的soc-sdk编译需要在SoC平台上运行的程序。


## 模型转换和量化

导出的模型需要编译成BModel才能在SOPHON TPU上运行，如果使用下载好的BModel可跳过本节。建议使用TPU-MLIR编译BModel。具体过程可参考算能官方文档：  
https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/tpu-mlir/quick_start/html/02_env.html

以下是一个模型转换和量化的具体例子。（命令均在docker /workspace环境下运行）

首先，需要准备__onnx模型文件__。如果使用tpu-mlir编译模型，则必须先将Pytorch模型导出为onnx模型。YOL0v8官方仓库提供了模型导出接口，可以直接使用它导出onnx模型：

```python
from ultralytics import YOLO
# Load a model
model = YOLO("yolov8s-pose.pt")  # 载入预训练模型
# Export the model
success = model.export(format="onnx", batch=1)  # 导出静态ONNX模型，需设置batch参数
```

上述脚本会在原始pt模型所在目录下生成导出的onnx模型，导出后可以修改模型名称以区分不同参数和输出类型，如`yolov8s_1b.onnx`表示单输出且输入batch大小为1的onnx模型。

__ONNX转MLIR__

`model_transform --model_name yolov8s-pose --model_def ../yolov8s-pose.onnx --mlir yolov8s-pose.mlir --input_shapes [[1,3,640,640]] --mean 0.0,0.0,0.0 --scale 0.0039216,0.0039216,0.0039216 --keep_aspect_ratio --pixel_format rgb --test_input ../image/pose_256_192.jpg --test_result yolov8s-pose_top_outputs.npz`

生成文件如下：
![Alt text](pics/image-1.png)

__MLIR转F16__

model_deploy --mlir yolov8s-pose.mlir --quantize F16 --processor bm1684x --model yolov8s-pose_1684x_f16.bmodel --test_input yolov8s-pose_in_f32.npz --test_reference yolov8s-pose_top_outputs.npz

生成文件如下：
![Alt text](pics/image-2.png)

__生成校准表__ 

run_calibration yolov8s-pose.mlir --dataset ../COCO2017 --input_num 100 -o yolov8s-pose_cali_table

生成文件 yolov8s-pose_cali_table，用于后续编译 INT8 模型的输入文件。

__MLIR转INT8__

model_deploy \
--mlir yolov8s-pose.mlir \
--quantize INT8 \
--calibration_table yolov8s-pose_cali_table \
--processor bm1684 \
--test_input yolov8s-pose_in_f32.npz \
--test_reference yolov8s-pose_top_outputs.npz \
--tolerance 0.85,0.45 \
--model yolov8s-pose_1684_int8_sym.bmodel

__模型性能测试__

`bm-smi`  
`bmrt_test --bmodel yolov8s-pose_1684x_f16.bmodel`
![Alt text](pics/image-3.png)

`bmrt_test --bmodel yolov8s-pose_1684x_int8_sym.bmodel`
![Alt text](pics/image-4.png)

其中，6,7行为bmodle网络输入输出信息； 倒数第4行为加载数据到 NPU 的 DDR 的时间，及加载数据到 NPU 的 DDR 的时间是指将数据从主机（例如 CPU 或 GPU）传输到 NPU（神经网络处理单元）的 DDR（动态随机存取内存）中的时间； 倒数第2行为输出数据取回时间


## 程序编译

C++程序运行前需要编译可执行文件。

### x86/arm PCIe平台
可以直接在PCIe平台上编译程序：

```bash
cd cpp/yolov8_bmcv
mkdir build && cd build
cmake .. 
make
cd ..
```
编译完成后，会在yolov8_bmcv目录下生成yolov8_bmcv.pcie。

### SoC平台
通常在x86主机上交叉编译程序，您需要在x86主机上使用SOPHON SDK搭建交叉编译环境，

```bash
cd cpp/yolov8_bmcv
mkdir build && cd build
#请根据实际情况修改-DSDK的路径，需使用绝对路径。
cmake -DTARGET_ARCH=soc -DSDK=/path_to_sdk/soc-sdk ..  
make
```
编译完成后，会在yolov8_bmcv目录下生成yolov8_bmcv.soc。


## 推理测试
对于PCIe平台，可以直接在PCIe平台上推理测试；对于SoC平台，需将交叉编译生成的可执行文件及所需的模型、测试数据拷贝到SoC平台中测试。测试的参数及运行方式是一致的，下面主要以PCIe模式进行介绍。

### 参数说明
可执行程序默认有一套参数，请注意根据实际情况进行传参，具体参数说明如下：
```bash
Usage: yolov8_bmcv.pcie [params]

        --bmodel (value:../../models/BM1684/yolov8s_fp32_1b.bmodel)
                bmodel file path
        --classnames (value:../../datasets/coco.names)
                class names file path
        --conf_thresh (value:0.25)
                confidence threshold for filter boxes
        --dev_id (value:0)
                TPU device id
        --help (value:true)
                print help information.
        --input (value:../../datasets/test)
                input path, images direction or video file path
        --nms_thresh (value:0.7)
                iou threshold for nms
```
**注意：** CPP传参与python不同，需要用等于号，例如`./yolov8_bmcv.pcie --bmodel=xxx`。

### 测试图片
图片测试实例如下，支持对整个图片文件夹进行测试。
```bash
./yolov8_bmcv.soc --input=../../datasets/test --bmodel=../../models/yolov8s-pose_1684x_f16.bmodel --conf_thresh=0.25 --nms_thresh=0.7 
```
测试结束后，会将预测的图片保存在`results/images`下，预测的结果保存在`results/yolov8s_fp32_1b.bmodel_test_bmcv_cpp_result.json`下，同时会打印预测结果、推理时间等信息。

### 测试视频
视频文件调用示例如下。RTSP 地址仅为参数格式示例；当前输入检查仍需处理上述已知问题，尚未验证其可用性。
```bash
./yolov8_bmcv.soc --input=../../datasets/dance_1080P.mp4  --bmodel=../../models/yolov8s-pose_1684x_f16.bmodel --conf_thresh=0.25 --nms_thresh=0.7

./yolov8_bmcv.soc --input=rtsp://USER:PASSWORD@CAMERA_HOST:554/STREAM_PATH --bmodel=../../models/yolov8s-pose_1684x_f16.bmodel  --dev_id=0 --conf_thresh=0.25 --nms_thresh=0.7 --classnames=../../datasets/coco.names
```
测试结束后，会生成并保存在`results/videos`中，同时会打印预测结果、推理时间、帧率等信息。