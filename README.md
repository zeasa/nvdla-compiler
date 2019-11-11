# nvdla-compiler-learning #

# 1.安装准备与一些有用的资源 #
## 1.1.github地址 ##
	https://github.com/nvdla
## 1.2.编译和安装 ##
### 1.2.1.参考信息 ###
	https://blog.csdn.net/zhajio/article/details/84784336 
	https://blog.csdn.net/hywCogost/article/details/82114529
### 1.2.2.问题与解决 ###

- ubuntu16.04编译会出现：“  undefined reference to google::protobuf::internal::empty_string_[abi:cxx11]   ”等链接错误，原因是ubuntu16.04默认安装的是GCC5，但是nvdla的sw部分应该是用的GCC5以下的版本，google上有人讲到：“  the ABI for std::string has changed in GCC 5(related to c++ 11 requirements, but it applies even if you aren't using c++ 11   ”，解决方法是：可以在g++的编译参数中加入 -D_GLIBCXX_USR_CXX11_ABI=0, 然后就解决了，具体修改文件是nvdla/sw/umd/core/src/compiler/Makefile 把上述的字符串加到MODULE_CPPFLAGS.....以后最末尾即可编译通过

## 1.3.软硬件分析文章参考 ##
	https://github.com/JunningWu/Learning-NVDLA-Notes
# 2.源代码结构 #
## 2.1.整体代码目录结构 ##
## 2.2.hw目录结构 ##
![](https://github.com/zeasa/nvdla-compiler/raw/master/document/imgs/hwfolderlist.png)

- cmod是systemc模型用来仿真和vp
- perf是性能计算excel表格
- spec是配置和工程文件
- sync是综合配置
- tool是build和pl脚本等工具
- verif是仿真文件夹
- vmod是verilog仿真模型和RTL代码
## 2.3.sw目录结构 ##
![](https://github.com/zeasa/nvdla-compiler/raw/master/document/imgs/swfolderlist.png)

- umd是runtime的上层部分，运行在用户态，负责解析loadable文件并提交给kmd驱动硬件执行计算任务
- kmd接受umd的工作负载提交，并驱动硬件DLA执行计算任务
- prebuild

# 3.模型编译与DLA仿真运行 #
# 4.深入源码 #
## 4.1.总体结构与代码的日志系统 ##

### 4.1.1.代码总体结构

​	nvdla的软件代码部分主要分为umd和kmd，这两部分的作用在前面sw目录结构部分已经说过。其中umd包括了runtime的userspace部分和compiler部分。umd文件夹包括了如下几个文件夹，下面说明其功能：

- apps：包括了runtime的入口以及compiler的入口
- core：runtime和compiler的主要实现逻辑放在这里，也是需要着重阅读的部分
- externel：这个目录存放了umd用到的第三方库，需要说明的是protobuf主要用于caffe模型的解析，因为caffe的blob格式原始存储方式是使用了google开发的protobuf
- make：umd的编译makefile
- port：主要是runtime的底层访问API，为了实现OS无关，做了这个隔离层，目前只实现了Linux下的。这层API包括内存访问，用户态和内核态的交互IOCTL，内存分配等。需要注意的是NVDLA的runtime部分用户态和内核态交互使用了Linux用于显卡抽线的DRM接口
- utils：这个文件夹放了几个通用的模块，包括BitBinaryTree以及伙伴内存管理等。其中伙伴内存管理模块在compiler的tensor内存分配推导部分有用到

### 4.1.2.日志系统

​	nvdla的sw部分，文档比较缺乏，在nvdia的官方网站只有半页简单介绍，关于软件框架，层次结构一概为止。代码里面注释也很少，只有在涉及部分算法的函数有很少的几行简单的说明。好在nvdla的sw软件代码里，有较为详细的log日志生成功能，可以将代码在编译的过程中的内部数据结构和变量很好的展示出来，在代码阅读过程中有很大的帮助，很多读不懂的部分，看看日志就能明白其中的联系。

​	nvdla的日志，在代码里默认都是关闭的，并且没有总体的开关，log开关都是分散在各个类的定义文件里。下面举个例子：

![](https://github.com/zeasa/nvdla-compiler/raw/master/document/imgs/logswitch.png)

上图中在Graph这个类里面，有许多的log日志开关，只需将红框中的false改为true就可以打开这个class的日志输出。类似的开关还有很多，需要在读到相关class部分代码的时候有需要的打开。以编译一个Lenet5的网络为例，输出的日志在10000行左右，这个log.txt在本repo的model&log文件夹里可以找到。

## 4.2.runtime部分 ##

### 4.2.1.总体概述

​	ToDo

### 4.2.2.结构和执行流程分析

​	ToDo

## 4.3.compiler部分 ##

### 4.3.1.总体概述

​	compiler部分的代码主要在sw/umd/core/src/compiler目录里，经过阅读，发现nvdla现有的compiler代码前端只支持caffe一种前端框架，在调用compiler进行模型编译的时候，命令行参数需要指定caffe模型的prototxt文件以及train好的model的部署文件(包括了weight和bias等参数)。caffe模型的prototxt文件格式具体可以参考caffe框架相关文档。以下是一个prototxt文件的一部分：

```js
name: "LeNet"
layer {
  name: "data"
  type: "Input"
  top: "data"
  input_param { shape: { dim: 64 dim: 1 dim: 28 dim: 28 } }
}
layer {
  name: "conv1"
  type: "Convolution"
  bottom: "data"
  top: "conv1"
  param {
    lr_mult: 1
  }
  param {
    lr_mult: 2
  }
  convolution_param {
    num_output: 20
    kernel_size: 5
    stride: 1
    weight_filler {
      type: "xavier"
    }
    bias_filler {
      type: "constant"
    }
  }
}
layer {
  name: "pool1"
  type: "Pooling"
  bottom: "conv1"
  top: "pool1"
  pooling_param {
    pool: MAX
    kernel_size: 2
    stride: 2
  }
}
```

从这个例子可以看出，同大多框架的网络模型定义相似，网络定义都是以layer为主，顺序定义，语法为JSON。每层包括了name，type和param等参数，其中layer的type以及每种type的layer的参数在caffe框架的定义文件里有详细的描述。上面这个例子只截取了一个LeNet5网络的前3层，分别是Input、conv1、pooling等。

​	接下来compiler会对prototxt定义的网络模型进行解析，生成内部的CanonicalAST数据结构，这部分在compiler目录下的AST.cpp和CanonicalAST.cpp两个文件里进行实现。但CanonicalAST只是一种过渡的表示，下面紧接着会执行从CanonicalAST到EngineAST的变换，后续的所有AST变换与优化都是针对于EngineAST进行的，感觉这个AST才是整个nvdla编译框架的中间IR表示。

​	EngineAST生成之后，compiler会对这个中间表示做各种变换与优化，这一步的结果就是要得到一个适合后端代码生成的AST表示。

​	最后一步就是根据变换和优化之后的EngineAST数据结构进行代码生成。这个阶段最终要的一项工作就是要解决tensor内存分配的问题，这个工作在memroyResolver阶段完成。

### 4.3.2.compiler总体执行流程

![](https://github.com/zeasa/nvdla-compiler/raw/master/document/imgs/compiler_flowchart.png)

### 4.3.3.代码流程分析-前端caffe模型到CanonicalAST表示

### 4.3.4.CanonicalAST到EngineAST表示

### 4.3.5.EngineAST中间IR变换与优化PASS

### 4.3.6.EngineAST到后端代码Emit（代码生成）