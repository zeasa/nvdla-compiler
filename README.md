# nvdla-compiler-learning #

[TOC]



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

### 4.3.2.compiler执行流程

![](https://github.com/zeasa/nvdla-compiler/raw/master/document/imgs/compiler_flowchart.png)

1. main()函数是compiler的入口，主要功能是处理compiler命令行参数以及调用launchTest()，下表列出命令行参数

   ```
   Usage: %s [-options] --prototxt <prototxt_file> --caffemodel <caffemodel_file>
   where options include:
   -h                                              print this help message
   -o <outputpath>                                 outputs wisdom files in 'outputpath' directory
   --profile <basic|default|performance|fast-math> computation profile (default: fast-math)
   --cprecision <fp16|int8>                          compute precision (default: fp16)
   --configtarget <opendla-full|opendla-large|opendla-small>   target platform (default: nv_full)
   --calibtable <int8 calib file>                  calibration table for INT8 networks (default: 0.00787)
   --quantizationMode <per-kernel|per-filter>      quantization mode for INT8 (default: per-kernel)
   --batch                                           batch size (default: 1)
   --informat <ncxhwx|nchw|nhwc>                     input data format (default: nhwc)
   ```
   从命令函参数可以看出，目前nvdla的compiler只支持caffe模型，量化精度支持INT8和fp16，并且可以支持multibatch

2. launchTest()这个函数调用testSetup函数，根据appArgs结构体，填充testInfo结构体，并以testInfo结构体为参数调用parseAndCompile()函数

   ```c
   TestInfo testInfo;
   PROPAGATE_ERROR_FAIL(testSetup(appArgs, &testInfo));
   PROPAGATE_ERROR_FAIL(parseAndCompile(appArgs, &testInfo));
   ```
   这里涉及到两个重要的结构体TestAppArgs和TestInfo

   ```c
   struct TestAppArgs
   {
       std::string inputPath;
       std::string inputName;
       std::string loadableName;
       NvS32 serverPort;
       NvU8 normalize_value;
       float mean[4];
       bool rawOutputDump;
   };
   struct TestInfo
   {
       // runtime
       nvdla::IRuntime* runtime;
       std::string inputLoadablePath;
       NvU8 *inputHandle;
       NvU8 *outputHandle;
       NvU8 *pData;
       bool dlaServerRunning;
       NvS32 dlaRemoteSock;
       NvS32 dlaServerSock;
       NvU32 numInputs;
       NvU32 numOutputs;
       NvDlaImage* inputImage;
       NvDlaImage* outputImage;
   };
   ```

3. testSetup()：主要是检查输入输出文件路径有效性，删除前一次编译中间文件，新建新一次编译中间文件夹

   ```c++
   NvDlaError testSetup(const TestAppArgs* appArgs, TestInfo* i)
   {
       NvDlaError e = NvDlaSuccess;
       std::string wisdomPath = appArgs->outputPath + "wisdom.dir/";
       std::string removeCmd = "";
       std::string imagePath = "";
       NvDlaStatType stat;
       int ii = 0;
   
       // Do input paths exist?
       e = NvDlaStat(appArgs->inputPath.c_str(), &stat);
       if (e != NvDlaSuccess)
           ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "Input path does not exist: \"%s\"", appArgs->inputPath.c_str());
   
       // Do output paths exist?
       e = NvDlaStat(appArgs->outputPath.c_str(), &stat);
       if (e != NvDlaSuccess)
           ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "Output path does not exist: \"%s\"", appArgs->outputPath.c_str());
   
       // Clear wisdomPath if any exist
       removeCmd += "rm -rf " + wisdomPath;
       ii = std::system(removeCmd.c_str()); // This is pretty awful
       if (ii != 0)
           ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "system command failed: \"%s\"", removeCmd.c_str());
   
       PROPAGATE_ERROR_FAIL(NvDlaMkdir(const_cast<char *>(wisdomPath.c_str())));
   
       // Initialize TestInfo
       i->wisdom = NULL;
       i->wisdomPath = wisdomPath;
       i->pData = NULL;
   
       return NvDlaSuccess;
   fail:
       return e;
   }
   ```

   parseAndCompiler()函数：

   ```c++
   NvDlaError parseAndCompile(const TestAppArgs* appArgs, TestInfo* i)
   {
       NvDlaError e = NvDlaSuccess;
       bool isCaffe = appArgs->caffemodel != "";
   
       PROPAGATE_ERROR_FAIL(parseSetup(appArgs, i));//这个函数为空，直接返回OK
   
       NvDlaDebugPrintf("creating new wisdom context...\n");
       i->wisdom = nvdla::createWisdom();//建立编译环境，这里这个wisdom是一个接口类，工厂类和工厂模式应用
       if (!i->wisdom)
           ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "createWisdom() failed");
   
       NvDlaDebugPrintf("opening wisdom context...\n");
       if (!i->wisdom->open(i->wisdomPath))
           ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "wisdom->open() failed to open: \"%s\"", i->wisdomPath.c_str());
   
       // Parse，这里这个函数负责parse caffemodel的两个输入文件
       if (isCaffe)
           PROPAGATE_ERROR_FAIL(parseCaffeNetwork(appArgs, i));
       else
           ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "Unknown network type encountered");
   
       // Compile
       PROPAGATE_ERROR_FAIL(compileProfile(appArgs, i));
   
       /* Destroy network before closing wisdom context */
       nvdla::destroyNetwork(i->wisdom->getNetwork());
   
       NvDlaDebugPrintf("closing wisdom context...\n");
       i->wisdom->close();
   fail:
       if (i->wisdom != NULL) {
           nvdla::destroyWisdom(i->wisdom);
           i->wisdom = NULL;
       }
       return e;
   }
   ```

4. parseCaffeNetwork()：这个函数负责解析命令行传递的编译输入model文件，包括prototxt和caffemodel，前者主要定义网络的结构和参数，后者包含train好的网络的weight和bias参数值，这里只贴出这个函数最重要的部分：

   ```c++
   static NvDlaError parseCaffeNetwork(const TestAppArgs* appArgs, TestInfo* i)
   {
       NvDlaError e = NvDlaSuccess;
       nvdla::INetwork* network = NULL;
       const nvdla::caffe::IBlobNameToTensor* b = NULL;
       nvdla::caffe::ICaffeParser* parser = nvdla::caffe::createCaffeParser();
       std::string caffePrototxtFile = appArgs->prototxt.c_str();//caffe模型的prototxt文件
       std::string caffeModelFile = appArgs->caffemodel.c_str();//caffe模型的caffemodel文件，blob格式
   	
       //这里创建网络的内存表示，主要涉及INetwork接口类和Network实现类，这里network的create使用了工厂模式
       network = nvdla::createNetwork();
       if (!network)
           ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "createNetwork() failed");
   	
       //parser->parse()函数负责caffe模型的解析，传递的参数是caffe模型的两个文件，输出是network类和IBlobNameTOTensor两个
       NvDlaDebugPrintf("parsing caffe network...\n");
       b = parser->parse(caffePrototxtFile.c_str(), caffeModelFile.c_str(), network);
       if (!b)
           ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "Unable to parse caffemodel: \"%s\"", caffePrototxtFile.c_str());
   }
   ```

   这个函数涉及了两个重要的数据结构：INetwork和Network，这里列出这两个数据结构的主要部分

   ```c++
   class INetwork
   {
   public:
       virtual ITensor* addInput(const char * name, Dims4 dimensions) = 0;
   
       //指定网络的Input和Output Tensor
       virtual bool markInput(ITensor * tensor) = 0;
       virtual void markOutput(ITensor * tensor) = 0;
   	
       //构建网络的API函数，理论上通过以下这组add函数，就可以不使用caffe模型，手工的创建一个网络，类似大多数框架提供的网络构造API函数，但NVDLA似乎没有对外开放这组接口用于手工构造网络，TVM框架就对望开放了这组接口
       virtual IConvolutionLayer *    addConvolution   (ITensor * input, int numOutputs, int paddingValue, Dims2 kernelSize,  Dims2 tlPadding, Dims2 brPadding, Dims2 stride, Dims2 dilation,
   Weights kernelWeights, Weights biasWeights, BiasMode biasMode, int numGroups) = 0;
       virtual IFullyConnectedLayer * addFullyConnected(ITensor * input, int outputSize, Weights kernelWeights, Weights biasWeights, BiasMode biasMode) = 0;
       virtual IActivationLayer *     addActivation    (ITensor * input, ActivationType type) = 0;
       virtual IPoolingLayer *        addPooling       (ITensor * input, PoolingType type,
    Dims2 windowSize, Dims2 stride, Dims2 tlPadding, Dims2 brPadding) = 0;
       virtual ILRNLayer *            addLRN           (ITensor * input, int window, float alpha, float beta, float k) = 0;
       virtual IScaleLayer *          addScale         (ITensor * input, ScaleMode mode, Weights shift, Weights scale, Weights power) = 0;
       virtual IBatchNormLayer *      addBatchNorm     (ITensor * input, BatchNormMode mode, Weights mean, Weights variance, float epsilon) = 0;
       virtual ISoftMaxLayer *        addSoftMax       (ITensor*input) = 0;
       virtual IConcatenationLayer *  addConcatenation (ITensor*const*inputs, int numInputs) = 0;
       virtual ISliceLayer *          addSlice         (ITensor*input, int numOutputs) = 0;
       virtual IDeconvolutionLayer *  addDeconvolution (ITensor * input, int numOutputs, int paddingValue, Dims2 kernelSize, Dims2 tlPadding, Dims2 brPadding, Dims2 stride, Dims2 dilation,
   Weights kernelWeights, Weights biasWeights, BiasMode biasMode, int numGroups) = 0;
       virtual IElementWiseLayer   *  addElementWise   (ITensor *input0, ITensor* input1, ElementWiseOperation op) = 0;
   
       virtual int getNumInputs()  const  = 0;
       virtual int getNumOutputs() const  = 0;
       virtual int getNumLayers()  const  = 0;
       virtual ILayer  * getLayer(int index)  const = 0;
       virtual ITensor * getOutput(int index) const = 0;
       virtual ITensor * getInput(int index)  const = 0;
       virtual void setPoolingOutputDimensionsFormula      (OutputDimensionsFormula* callback) = 0;
       virtual void setConvolutionOutputDimensionsFormula  (OutputDimensionsFormula* callback) = 0;
       virtual void setDeconvolutionOutputDimensionsFormula(OutputDimensionsFormula* callback) = 0;
       virtual OutputDimensionsFormula& getPoolingOutputDimensionsFormula()       const = 0;
       virtual OutputDimensionsFormula& getConvolutionOutputDimensionsFormula()   const = 0;
       virtual OutputDimensionsFormula& getDeconvolutionOutputDimensionsFormula() const = 0;
       virtual const std::vector<ITensor *> & getInputs()  const = 0;
       virtual const std::vector<ILayer * > & getLayers()  const = 0;
       virtual const std::vector<ITensor *> & getOutputs() const = 0;
   };
   ```

   INetwork接口类是一个纯虚类，仅作接口使用，不可以被实例化，可以被继承，这个接口是NVDLA所谓的中间IR的入口。parseCaffeNetwork()函数调用了createNetwork()函数创建了一个Network的实例，这里使用了类工厂模式。

5. compile()

6. compilerInternal()

7. compilerInternal()

8. generateGraph(), generateGraph(), emit()

### 4.3.3.代码流程分析-前端caffe模型到CanonicalAST表示

### 4.3.4.代码流程分析-CanonicalAST到EngineAST表示

### 4.3.5.代码流程分析-EngineAST中间IR变换与优化PASS

### 4.3.6.代码流程分析-EngineAST到后端代码Emit（代码生成）