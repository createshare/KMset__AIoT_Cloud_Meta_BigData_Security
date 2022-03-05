# OpenCV 计算机视觉库

OpenCV（Open Source Computer Vision Library）是一个基于 BSD 许可（开源）发行、跨平台（支持 Linux、Windows、Android、MacOS）、轻量且高效的计算机视觉库，提供了处理图像和视频的能力。

OpenCV 于 1999 年由 Gary Bradsky 在英特尔创立，第一个版本于 2000 年问世。

OpenCV 主要由 C 语言和 C++ 语言来编写的，它的主要接口也是 C++ 语言，但是依然保留了大量的 C 语言接口。支持各种编程语言，轻量级且高效——由一系列 C 函数和少量 C++ 类构成同时提供了 Python、Ruby、MATLAB 等语言的接口，实现了图像处理和计算机视觉的很多通用算法。



## OpenCV 特点

- OpenCV采用C/C++语言编写，可以运行在Linux/Windows/Mac等操作系统上。
- OpenCV提供了Python、Ruby、MATLAB以及其他语言的接口。
- OpenCV-Python 是 OpenCV 的 Python API，它结合了 OpenCV C++ API 和 Python 语言的最佳特性。
- 它采用优化的C代码编写，能够充分利用多核处理器的优势。
- 基于 CUDA 和 OpenCL 的高速 GPU 操作接口也在积极开发中。
- 执行速度尽量快，主要关注实时应用。
- 具有良好的可移植性



# OpenCV 模块组成

OpenCV 具有模块化结构，其开发包里面包含多个共享库和静态库。

## 核心功能 cxcore

核心功能模块，是一个紧凑、小巧而高效的模块，定了基本的数据结构，包括密集的多维 Mat 数组（多尺度向量矩阵）和被其他模块使用的基本函数。CXCore 包含 OpenCV 的一些基本数据结构和相关函数。

- 基础结构：CvPoint,CvSize,CvScalar等
- 数组操作：cvCreateImage,cvCreateMat等
- 动态结构：CvMemStorage,CvMemBlock等
- 绘图函数：cvLine,cvRectangle等
- 数据保存和运行时类型信息：CvFileStorage,cvOpenFileStorage等
- 错误处理和系统函数:cvGetErrStatus,cvAlloc,cvFree等

## Machine Learning(ML)

ML 是机器学习库，包含一些基于统计的分类和聚类工具。

包含许多聚类、分类和数据分析函数。如Bayes分类器，K近邻算法，支持向量机，决策树，神经网络等等。

## CV 模块

OpenCV 的 CV 模块包含基本的图像处理函数和高级的计算机视觉算法。

### 图像处理

图像处理模块，包括线性和非线性图像滤波、几何图像转化（图像缩放、仿射变换、透视矫正、通用的基于表格的像素映射）、色域空间转换、直方图生成与分析等。

图像处理：cvSobel，cvCanny等

### 视频分析

视频分析模块，包括运动检测、动作判断、背景弱化、目标跟踪算法等。

### 3D 校准（calib3d）

基于多视图的几何算法，可以实现平面和立体摄像机校准、对象姿势判断、立体匹配算法、3D  元素的重建等。

摄像头定标与三维重建：cvCalibrateCamer2

### 平面特征（Features2D）

图像显著特征检测、特征描述、特征匹配等。

- 结构分析：ContourArea等

### 对象侦查（Objdetect）

对象检测和预定义类别实例化的侦查，例如脸、眼睛、杯子、人、汽车等。

- 运动分析与目标跟踪：cvMeanShift等

- 模式识别：CvHaarFeature

## HighGUI 

HighGUI 包含图像和视频输入/输出的函数。提供了比较容易使用的 UI 接口。

- 图像界面函数： cvNamedWindow
- 读图像和保存图像：cvLoadImage，cvSaveImage
- 读视频和写视频：CvCreateFileCapture等

## 视频输入输出（Video I/O）

提供了基本的视频采集、存取访问、编解码功能。

## GPU

为不同的 OpenCV 算法模块，提供 GPU 加速算法。

## 其他

例如，FLANN 和 Google 测试封装层、Python 绑定等。





# OpenCV 应用领域

1、人机互动
2、物体识别
3、图像分割
4、人脸识别
5、动作识别
6、运动跟踪
7、机器人
8、运动分析
9、机器视觉
10、结构分析
11、汽车安全驾驶

# OpenCV 源代码

## OpenCV 源代码主要模块：

- OpenCV core 核心库：为 OpenCV 的主要算法来源。核心库提供了很多组件，包括：
  - opencv_contrib 代码仓库：主要用于管理新功能模块的开发。
  - opencv_extra 代码仓库：存放了运行测试和示例程序时需要使用的一些测试数据和脚本。

