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

- **tools/bin/tmake -build cmod_top - can't build cmod_top**

  一般在Ubuntu14.04（gcc, g++ 4.8.4), 16.04 (gcc, g++ 5.4.0)可以顺利编译通过，在18.04 (gcc, g++ 8.x)以上会编译报错。因为hw支持的是低版本gcc, g++，如果要加入对高版本gcc，g++的支持，可以在**cmod/hls/include 目录下更新Algorithmic C**。

  详细见 [Fix CMOD Makefile calling system GCC linker instead of user GCC #191](https://github.com/nvdla/hw/pull/191)

  ```shell
  # 笔者解决方案
  cd PATH_TO_CMOD_HLS_INCLUDE
  git clone https://github.com/hlslibs/ac_types tmp
  cp tmp/include/* ./
  rm -rf tmp
  ```
- **$ cmake部分出现-Could NOT find Lua**

  ``` -- Searching for SystemC
  -- SystemC version = 2.3.0
  -- SystemC library = /usr/local/systemc-2.3.0/lib-linux64/libsystemc.so
  -- Searching for TLM
  running ls /usr/local/systemc-2.3.0/include/tlm.h 2>&1
  /usr/local/systemc-2.3.0/include/tlm.h
  -- TLM library = /usr/local/systemc-2.3.0/include/tlm.h
  CMake Error at /usr/share/cmake-3.10/Modules/FindPackageHandleStandardArgs.cmake:137 (message):
  Could NOT find Lua (missing: LUA_LIBRARIES LUA_INCLUDE_DIR)
  Call Stack (most recent call first):
  /usr/share/cmake-3.10/Modules/FindPackageHandleStandardArgs.cmake:378 (_FPHSA_FAILURE_MESSAGE)
  cmake/FindLua.cmake:113 (FIND_PACKAGE_HANDLE_STANDARD_ARGS)
  CMakeLists.txt:55 (find_package)
  ```
  原因是缺少Lua5.2相关脚本环境。$ sudo apt-get install liblua5.2即可。
  

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

​	caffemodel文件是训练后带有网络权重信息的模型文件，使用的是google的protobuf格式二进制存储，使用caffemodel2json工具可以将其解析为json格式以便查看，下面是对应上述prototxt文件lenet5网络的json结果的一部分：

```js
dump
{
  "name": "LeNet",
  "layer": [ 
    { #这一层是LeNet的输入层
      "name": "mnist",
      "type": "Data",
      "top": [
        "data",
        "label"
      ],
      "include": [
        {
          "phase": 0 #表明当前层只有TRAIN阶段才包含进来
        }
      ],
      "phase": 0, #训练或者测试阶段 TRAIN=0, TEST=1
      "transform_param": {
        "scale": 0.00390625
      },
      "data_param": {
        "source": "examples/mnist/mnist_train_lmdb",
        "batch_size": 64,
        "backend": 1
      }
    },
    { #这一层是LeNet紧接着输入层的第一个conv层
      "name": "conv1",
      "type": "Convolution",
      "bottom": [
        "data"
      ],
      "top": [
        "conv1"
      ],
      "param": [
        {
          "lr_mult": 1.0 #weight的学习率
        },
        {
          "lr_mult": 2.0 #bias的学习率
        }
      ],
      "blobs": [ #这里的blobs存储的是这一层的卷积kernel的weight和bias信息
        { #blobs[0]应该是weight权重，从shape看出有20个kernel，每个5*5*1
          "data": [
            0.17512507736682892,
            0.20436875522136688,
            0.056398797780275345,
            0.005825345404446125,
            0.23611973226070404,
            "(495 elements more)"
          ],
          "shape": {
            "dim": [
              20, #N
              1,  #C
              5,  #H
              5   #W
            ]
          }
        },
        { #blobs[1]应该是bias值，shape是20
          "data": [
            -0.05203453078866005,
            -0.26182013750076294,
            -0.1220993623137474,
            -0.07315845042467117,
            0.002272228477522731,
            "(15 elements more)"
          ],
          "shape": {
            "dim": [
              20
            ]
          }
        }
      ],
      "phase": 0,
      "convolution_param": {
        "num_output": 20,
        "kernel_size": [
          5
        ],
        "stride": [
          1
        ],
        "weight_filler": {
          "type": "xavier"  #权值初始化方法
        },
        "bias_filler": {
          "type": "constant" #权值初始化方法
        }
      }
    },
```

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

2. launchTest()


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
   
       //删除整个wisdom文件夹，这个wisdom文件夹里面放了什么文件？？
       removeCmd += "rm -rf " + wisdomPath;
       ii = std::system(removeCmd.c_str()); // This is pretty awful
       if (ii != 0)
           ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "system command failed: \"%s\"", removeCmd.c_str());
   	
       //建立wisdomdir
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
   
       // Compile，下层编译实际工作
       PROPAGATE_ERROR_FAIL(compileProfile(appArgs, i));
   
       //释放network内存数据结构
       nvdla::destroyNetwork(i->wisdom->getNetwork());
   
       NvDlaDebugPrintf("closing wisdom context...\n");
       i->wisdom->close();
   fail:
       if (i->wisdom != NULL) {
           nvdla::destroyWisdom(i->wisdom); //释放wisdom数据结构
           i->wisdom = NULL;
       }
       return e;
   }
   
   NvDlaError compileProfile(const TestAppArgs* appArgs, TestInfo* i)
   {
       NvDlaError e = NvDlaSuccess;
       std::string profileName = "";
       std::string targetConfigName = "";
   
       NvDlaFileHandle file = 0;
       std::string fileName = "";
       NvU8 *buffer = 0;
       NvU64 size = 0;
   
       nvdla::ICompiler* compiler = i->wisdom->getCompiler();
       if (!compiler)
           ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "wisdom->getCompiler() failed");
   
       if (!(appArgs->configtarget != ""))
           ORIGINATE_ERROR_FAIL(NvDlaError_NotInitialized, "No target config found to load");
   
       targetConfigName = appArgs->configtarget;
   
       // Determine profile
       PROPAGATE_ERROR_FAIL(generateProfile(appArgs, &profileName, i));
   
       // 调用compiler的compiler函数执行实际编译动作
       NvDlaDebugPrintf("compiling profile \"%s\"... config \"%s\"...\n", profileName.c_str(), targetConfigName.c_str());
       PROPAGATE_ERROR_FAIL(compiler->compile(profileName.c_str(), targetConfigName.c_str(), &i->compiledLoadable));
   
       // 获取loadable数据结构size
       PROPAGATE_ERROR_FAIL(compiler->getLoadableImageSize(profileName.c_str(),
                                                       &size));
       if (size == 0) {
           ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter,
                               "Invalid size for a loadable");
       }
   	//分配内存，存放loadable的数据
       buffer = (NvU8 *) NvDlaAlloc(size);
       if (buffer == NULL) {
           ORIGINATE_ERROR_FAIL(NvDlaError_InsufficientMemory,
                               "Failed to allocate buffer for loadable");
       }
       //拷贝loadable数据，并把数据串列输出到nvdla文件
       PROPAGATE_ERROR_FAIL(compiler->getLoadableImage(profileName.c_str(), buffer));
       fileName = profileName + ".nvdla";
       PROPAGATE_ERROR_FAIL(NvDlaFopen(fileName.c_str(), NVDLA_OPEN_WRITE, &file));
       PROPAGATE_ERROR_FAIL(NvDlaFwrite(file, buffer, size));
   fail:
       NvDlaFclose(file);
       if (buffer != NULL) NvDlaFree(buffer);
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

   对于caffemodel的具体解析在parse()函数里实现，后面章节会具体的详解，这个函数涉及了两个重要的数据结构：INetwork和Network，这里列出这两个数据结构的主要部分

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
       //注意这三个接口函数，获取Network的输入tensors、输出tensors和层，返回是vector
       virtual const std::vector<ITensor *> & getInputs()  const = 0;
       virtual const std::vector<ILayer * > & getLayers()  const = 0;
       virtual const std::vector<ITensor *> & getOutputs() const = 0;
   };
   ```
   ```c++
   INetwork *createNetwork()
   {
       priv::NetworkFactory::NetworkPrivPair n = priv::NetworkFactory::newNetwork();
       return n.i();
   }
   
   class NetworkFactory
   {
   public:
       typedef PrivPair<INetwork *, Network*> NetworkPrivPair;
   	
       //类工厂模式，注意，以下这些函数必须是static类型
       static NetworkPrivPair newNetwork();
       static NvDlaError deleteNetwork(INetwork *network);
   
       static Network *priv(INetwork *);//通过INetwork查找关联的Network
       static INetwork *i(Network *); //通过Network查找关联的INetwork
       static INetwork *self(void *s);
   
       static INetwork *deserializeFrom(WisdomContainerEntry *);
   
   protected:
       static BiMap<INetwork *, Network *> s_priv; //BiMap双向映射数据结构方便前后两个数据相互查找
       static BiMap<void *, INetwork *> s_self; //BiMap双向映射数据结构
   
       static INetwork *deserializeNetwork(WisdomContainerEntry *);
   };
   NetworkFactory::NetworkPrivPair NetworkFactory::newNetwork()
   {
       INetwork *network;
       Network *network_priv;
       network = network_priv = new priv::Network();//实际创建的是Network类型
       if (network) {
           s_priv.insert(network, network_priv);
           s_self.insert(network, network);
       }
       return NetworkPrivPair(network, network_priv);
   }
   
   // PrivPair and PrivDiamond simplify management of the pointers necessary
   // to track public interfaces, their private implementations and derivations
   // of such which result in a diamond inheritance pattern.  These are simply
   // fancy 2 and 4-tuples implemented by std::pair and 2x same.
   // Note: with RTTI enabled this can all disappear as dynamic_cast<>()
   // would be available instead ;(
   //这个模板类实现了一个Interface类和他的一个具体实现之间相互关联的数据结构，这么做应该是为了
   //实现RTTI功能
   template <typename I, typename P>
   class PrivPair
   {
   public:
       typedef I InterfaceType;
       typedef P PrivateType;
   
       PrivPair() : m_i_priv(0, 0) { }
       PrivPair(I i, P priv) :
           m_i_priv(i, priv) { }
       PrivPair(const PrivPair &p) :
           m_i_priv(p.m_i_priv) { }
       inline bool operator !() const { return (!m_i_priv.first) || (!m_i_priv.second); }
       inline bool operator ==(const PrivPair &rhs) const { return m_i_priv == rhs.m_i_priv; }
       inline bool operator <(const PrivPair &rhs) const { return m_i_priv < rhs.m_i_priv; }
       inline I i() const      { return m_i_priv.first;  }
       inline P priv() const   { return m_i_priv.second; }
   protected:
       std::pair<I, P> m_i_priv;
   };
   ```

5. compile()

   ```c++
   //这个函数接受的参数包括，profileName，targetConfigName，ILoadable双重指针
   NvDlaError Compiler::compile(const char *tp_name, const char *target_config_name, ILoadable **peli)
   {
       NvDlaError e = NvDlaSuccess;
       //调用compileInternal()函数完成实际编译工作
       CATCH_PROPAGATE_ERROR_FAIL(
           compileInternal(tp_name, target_config_name, peli, true /*full compile*/)
       );
   fail:
       return e;
   }
   ```

   这个函数实际调用了compileInternal()函数完成实际编译工作，但涉及到了一个重要的数据接口类:ILoabable

   ```c++
   class ILoadable
   {
   public:
       enum Interface;
       enum MemoryDomain;
       enum MemoryFlags;
       enum EventOp;
       
       //以下这些struct定义了loadable文件中的一系列重要的数据结构，
       //compiler的核心功能就是把模型编译成下面这些数据结构存入loadable文件
       //runtime的核心功能就是从loadable中解析如下数据结构并提交硬件进行计算
       struct Version;
       struct MemoryListEntry;
       struct EventListEntry;
       struct TaskListEntry;
       struct SubmitListEntry;
       struct AddressListEntry;
       struct TensorDescListEntry;
       struct RelocEntry;
       struct Blob;
   
       virtual std::string getName() const = 0;
       
       virtual int getNumMemoryListEntries() const = 0;
       virtual MemoryListEntry getMemoryListEntry(NvU16 mem_id) const = 0;
       
       virtual int getNumEventListEntries() const = 0;
       virtual EventListEntry getEventListEntry(NvU16 event_id) const = 0;
       
       virtual int getNumTaskListEntries() const = 0;
       virtual TaskListEntry getTaskListEntry(NvU16 task_id) const = 0;
       
       virtual int getNumAddressListEntries() const = 0;
       virtual AddressListEntry getAddressListEntry(NvU16 i) const = 0;
       
       virtual int getNumTensorDescListEntries() const = 0;
       virtual TensorDescListEntry getTensorDescListEntry(NvU16 i) const = 0;
       
       virtual NvDlaError getNetworkDataType(DataType::UnderlyingType *) const = 0;
       
       virtual NvDlaError getNumInputTensors(int *) const = 0;
       virtual NvDlaError getInputTensorDesc(NvU16 id, ILoadable::TensorDescListEntry *) const = 0;
       
       virtual NvDlaError getNumOutputTensors(int *) const = 0;
       virtual NvDlaError getOutputTensorDesc(NvU16 id, ILoadable::TensorDescListEntry *) const = 0;
   protected:
       ILoadable();
       virtual ~ILoadable();
   };
   ```

   在ILoadable接口中列出的一系列struct很重要，穿插在整个compiler工作的各个环节，后面会专门整理出来。

6. compilerInternal()

   第一层compilerInternal()函数，接收compiler函数传递过来的profile_name和target_config_name字符串，把这两个参数转换成Profile对象和TargetConfig对象，便于下一层compilerInternal函数使用：

   ```c++
   NvDlaError Compiler::compileInternal(const char *tp_name, const char *target_config_name, ILoadable **peli, bool fullCompile)
   {
       NvDlaError e = NvDlaSuccess;
       Profiler *profiler = 0;
       ProfileFactory::ProfilePrivPair p_profile;
       Profile *profile = 0;
       TargetConfig *target_config = 0;
       vector<engine_ast::Graph *> g;
   
       if ( !m_wisdom ) ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "No wisdom available.");
   
       profiler = ProfilerFactory::priv(m_wisdom->getProfiler());
       if ( !profiler ) ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "No profiler available.");
   
       //将tp_name字符串参数转换成Profile对象
       profile = ProfileFactory::priv(profiler->getProfile(tp_name));
       if ( !profile )
       {
           ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "Couldn't find profile to compile.");
       }
   	//将target_config_name字符串参数转换成TargetConfig对象
       target_config = TargetConfigFactory::priv(profiler->getTargetConfig(target_config_name));
       if ( !target_config )
       {
           ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "Couldn't find target config to compile.");
       }
   	//调用重载的compileInternal()执行下一步编译，这里参数已经是profile和target_config对象了
       PROPAGATE_ERROR_FAIL( compileInternal(profile, target_config, peli, fullCompile) );
   fail:
       return e;
   }
   ```

   上述代码涉及到两个重要的数据结构，Profile和TargetConfig类：

   ```c++
   class Profile : public IProfile
   {
   public:
       struct GlobalParams
       {
           NvU32                   m_NwInPixelOffX;
           NvU32                   m_NwInPixelOffY;
           nvdla::DataFormat       m_NwInDataFormat;     // NCHW default
           nvdla::DataFormat       m_NwOutDataFormat;    // NCHW default
           surface::SurfaceFormat  m_NwInSurfFormat;
           surface::SurfaceFormat  m_NwOutSurfFormat;
           surface::PixelMapping   m_NwInPixelMapping;
       };
       struct CompileParams
       {
           bool    m_canCompressWeights;
           bool    m_canWinograd;
           NvU32   m_CONVWeightBanksAllotted;
           NvU32   m_CONVDataBanksAllotted;
           bool    m_canSDPPDPOnFly;
           bool    m_canSDPMergeMathOps;
           bool    m_canSDPFuseSubEngineOps;
           bool    m_canSDPBustNOPs;
           bool    m_canSDPFuseVerticalOps;
           bool    m_useCVSRAMAllocate;
           bool    m_useMemPool;
           bool    m_useReusePooledMemory;
           bool    m_copyOutDebugSurfaces;
           bool    m_useGreedyEviction;
           NvU64   m_globalDRAMSize;
           NvU64   m_localDRAMSize;
           NvU64   m_localCVSRAMSize;
           NvU32   m_multiBatchSize;
           bool    m_canIMGPostChnlExtend;
           surface::SurfacePrecision m_computePrecision;
           nvdla::TensorScalingMode  m_tensorScalingMode;
           nvdla::QuantizationMode   m_quantizationMode;
       };
   protected:
       std::string m_name;
       std::map< std::string, ILoadable *> m_loadablesByName;
       std::vector<ILoadable *> m_loadables;
   
       GlobalParams m_globalParams;
       CompileParams m_compileParams;
   };
   ```

   这个Profile类主要是记录编译器的各种编译选项，其中有一部分应该是从命令行参数传递过来的。

   ```c++
   class TargetConfig : public ITargetConfig
   {
   public:
       struct TargetConfigParams
       {
           NvU32   m_atomicCSize;
           NvU32   m_atomicKSize;
           NvU32   m_memoryAtomicSize;
           NvU32   m_numConvBufBankAllotted;
           NvU32   m_numConvBufEntriesPerBank;
           NvU32   m_numConvBufEntryWidth;
           NvU32   m_maxBatchSize;
           bool    m_isWinogradCapable;
           bool    m_isCompressWeightsCapable;
           bool    m_isBatchModeCapable;
           bool    m_isPDPCapable;
           bool    m_isCDPCapable;
           bool    m_isSDPBiasCapable;
           bool    m_isSDPBatchNormCapable;
           bool    m_isSDPEltWiseCapable;
           bool    m_isSDPLutCapable;
           bool    m_isBDMACapable;
           bool    m_isRubikCapable;
       };
   protected:
       std::string m_instance_name;
       TargetConfigParams m_targetConfigParams;
   };
   ```

   这个TargetConfig数据结构主要用来记录DPU的内部硬件配置信息。

7. compilerInternal()

    这个函数是整个编译器的核心部分，主要包括了caffe模型到内部表示IR的转换，IR的各种优化变换，IR到后端代码生成等，后面会详细说明内部执行流程。

8. canonical_ast::generateGraph(), engine_ast::generateGraph(), emit()

    canonical_ast::generateGraph()功能是caffe模型到内部graph的变换，engine_ast::generateGraph()功能是内部graph到适配DPU的op的内部graph变换，emit()功能是后端代码生成，这三个函数是compilerInternal()的核心部分，剩余的其他函数主要执行graph的各种变换与优化，后面会详细分析。

### 4.3.3.代码流程分析-前端caffe模型到network内部表示

​	这部分功能实现在sw\umd\core\src\compiler\caffe\CaffeParser.cpp文件的CaffeParser::parse()函数当中。首先是几个数据结构：

```c++
class BlobNameToTensor : public IBlobNameToTensor
{
public:
    virtual void add(const std::string& name, ITensor* tensor);
    virtual ITensor* find(const char* name) const;
    virtual ITensor*& operator[](const std::string& name);
    virtual void setTensorNames();
    virtual ~BlobNameToTensor();
private:
    std::map<std::string, ITensor*> mMap;//proto文档当中的blob数据名称到Tensor的映射Map
};
//这个数据结构用来描述proto文件当中的blob数据
class BinaryProtoBlob : public IBinaryProtoBlob
{
public:
    BinaryProtoBlob(void* memory, DataType type, Dims4 dimensions);
    const void*	getData();
    Dims4 getDimensions();
    void	destroy();
protected:
    void* mMemory;//blob数据的实际内存地址
    DataType mDataType;//blob里存放的数据类型:FP32,FP16,INT16,INT8,UINT8,UINT16等
    Dims4 mDimensions;//数据格式NCHW等
    virtual ~BinaryProtoBlob();
};
class CaffeParser : public ICaffeParser
{
public:
    CaffeParser() : ICaffeParser(), mDeploy(NULL), mModel(NULL), mTmpAllocs(), mDimsCallback(NULL),
        mBlobNameToTensor(NULL), mProtobufBufferSize(1024 << 20)
    { }
    virtual const IBlobNameToTensor* parse(const char* deploy,
                                           const char* model,
                                           INetwork* network);
    virtual int identifyOutputs(INetwork * network);
    virtual ~CaffeParser();
    void setProtobufBufferSize(size_t size) { mProtobufBufferSize = size; }
    // read a blob from a protobuf file (typically a mean blob)
    static BinaryProtoBlob* parseBinaryProto(const char* fileName);
    static void shutdownProtobufLibrary();
private:
    ditcaffe::NetParameter * mDeploy;//
    ditcaffe::NetParameter * mModel;
    std::vector<void*> mTmpAllocs;
    INetwork::OutputDimensionsFormula* mDimsCallback;
    IBlobNameToTensor* mBlobNameToTensor;
    size_t mProtobufBufferSize;
};
```

要理解Caffemodel的parse，就需要了解caffe的model文件格式。前面讲了compiler的输入caffe文件包括了prototxt文件和caffemodel文件，其中prototxt文件时JSON格式的文本，主要描述了caffe网络的层次结构，那么caffemodel文件主要是存储了pre_trained的网络weight和bias参数信息。其中caffemodel文件是google的protobuf格式，其解析需要使用到protobuf库来进行。所有关于caffe模型解析的文件都位于sw/umd/core/src/compiler/caffe/目录下，其中此目录下的CaffeParser.cpp是caffemodel的解析器，其功能是调用ditcaffe文件夹下的文件完成的。ditcaffe文件夹下的ditcaffe.proto是caffemodel文件的proto结构定义文件，通过protobuf的编译器编译成protobuf-2.6.1目录下的ditcaffe.pb.cpp和ditcaffe.pb.h两个文件，即是实际caffemodel文件解析功能的具体实现。注意protobuf库解析过的caffemodel的内存变量格式是NetParameter类型，这个类型的实际定义来源于ditcaffe.proto文件定义。

```protobuf
syntax = "proto2";
//option optimize_for = LITE_RUNTIME;
package ditcaffe;
// Specifies the shape (dimensions) of a Blob.
message BlobShape {
  repeated int64 dim = 1 [packed = true];
}
message BlobProto {
  optional BlobShape shape = 7;
  repeated float data = 5 [packed = true];
  repeated float diff = 6 [packed = true];
  repeated double double_data = 8 [packed = true];
  repeated double double_diff = 9 [packed = true];
  repeated uint32 half_data = 10 [packed = true];
  repeated uint32 half_diff = 11 [packed = true];

  // 4D dimensions -- deprecated.  Use "shape" instead.
  optional int32 num = 1 [default = 0];
  optional int32 channels = 2 [default = 0];
  optional int32 height = 3 [default = 0];
  optional int32 width = 4 [default = 0];
}
```

上面这段代码就是ditcaffe.proto文件的开头部分，主要是定义了caffemodel文件里的数据存放格式。

```c++
const IBlobNameToTensor* CaffeParser::parse(const char* deployFile,const char* modelFile,
                                            INetwork * network)
{
    CHECK_NULL_RET_NULL(deployFile);
    CHECK_NULL_RET_NULL(modelFile);
    assert(mDimsCallback == 0);

    if (!mDimsCallback) {
        mDimsCallback = new CaffeParserPoolingDimsCallback;
    }
    network->setPoolingOutputDimensionsFormula(mDimsCallback);

    //调用readBinaryProto()函数解析modelFile文件，返回到NetParameter类型的mModel变量当中
    //modelFile=lenet_iter_10000.caffemodel
    //ditcaffe::NetParameter * mModel，这个NetParameter数据结构是从ditcaffe.proto自动生成的
    mModel = new dc::NetParameter();
    if (!readBinaryProto(mModel, modelFile, mProtobufBufferSize)) {
        gLogError << "Could not parse model file" << std::endl; return 0;
    }
    
    // There are some challenges associated with importing caffe models. One is that
	// a .caffemodel file just consists of layers and doesn't have the specs for its
	// input and output blobs.So we need to read the deploy file to get the input
	//readTextProto()函数解析deployFile文件，返回到NetParameter类型的mDeploy变量当中
    //deployFile=lenet.prototxt
    //ditcaffe::NetParameter * mDeploy，这个NetParameter数据结构是从ditcaffe.proto自动生成的
    mDeploy = new dc::NetParameter();
    if (!readTextProto(mDeploy, deployFile)) {
        gLogError << "Could not parse deploy file" << std::endl; return 0;
    }

    bool ok = true;
    //提取mModel中的weight数据，放到weights变量中，后面调用每层解析函数时作为参数传递进去
    CaffeWeightFactory weights(*mModel, false, mTmpAllocs);
	//mBlobNameToTensor变量维护一个blob文件中weight数据到ITensor*的映射关系
    mBlobNameToTensor = new BlobNameToTensor();
	
    //input blob只在prototxt文件当中，所以这里以mDeploy为基础给network增加inputTensor
    for (int i = 0; i < mDeploy->input_size(); i++) {
        Dims4 dims;
        if (mDeploy->input_shape_size()) {
            dims.n = (int)mDeploy->input_shape().Get(i).dim().Get(0);
            dims.c = (int)mDeploy->input_shape().Get(i).dim().Get(1);
            dims.h = (int)mDeploy->input_shape().Get(i).dim().Get(2);
            dims.w = (int)mDeploy->input_shape().Get(i).dim().Get(3);
        }
        else { // deprecated, but still used in a lot of networks
            dims.n = (int)mDeploy->input_dim().Get(i * 4 + 0);
            dims.c = (int)mDeploy->input_dim().Get(i * 4 + 1);
            dims.h = (int)mDeploy->input_dim().Get(i * 4 + 2);
            dims.w = (int)mDeploy->input_dim().Get(i * 4 + 3);
        }
        //调用network的API增加network的一个InputTensor
        //这里加入一个tensor只需要指定tensor的name和dims即可
        ITensor* tensor = network->addInput(mDeploy->input().Get(0).c_str(), dims);
        //建立network中新增InputTensor到blob文件中相应区域的name的cstring映射
        mBlobNameToTensor->add(mDeploy->input().Get(0), tensor);
    }
    
    //前面通过readBinaryProto()函数和readTextProto()函数把caffe模型的信息和weight等解析到NetParameter
  	//类型的变量mModel和mDeploy，这里通过对layer的迭代，通过NetParameter里的LayerParameter进行解析
    //逐步建立network的内存中间表示
    for (int i = 0; i < mDeploy->layer_size() && ok; i++) {
        const dc::LayerParameter& layerMsg = mDeploy->layer(i);
        if (layerMsg.has_phase() && layerMsg.phase() == dc::TEST) {
            continue;
        }
		//Dropout层处理
        if (layerMsg.type() == "Dropout")
        {
            mBlobNameToTensor->add(layerMsg.top().Get(0),
                                   mBlobNameToTensor->find(layerMsg.bottom().Get(0).c_str()));
            continue;
        }
		//Input层处理
        if (layerMsg.type() == "Input")
        {
            const dc::InputParameter& p = layerMsg.input_param();
            for (int i = 0; i < layerMsg.top_size(); i++)
            {
                const dc::BlobShape& shape = p.shape().Get(i);
                Dims4 dims(shape.dim().Get(0), shape.dim().Get(1), shape.dim().Get(2), shape.dim().Get(3));
                //调用network的API，增加Input层
                ITensor* tensor = network->addInput(layerMsg.top(i).c_str(), dims);
                mBlobNameToTensor->add(layerMsg.top().Get(i), tensor);
            }
            continue;
        }
        //Flatten层处理
        if (layerMsg.type() == "Flatten")
        {
            ITensor* tensor = (*mBlobNameToTensor)[layerMsg.bottom().Get(0)];
            (*mBlobNameToTensor)[layerMsg.top().Get(0)] = tensor;
            std::cout << "Warning: Flatten layer ignored." << std::endl;
            continue;
        }
	
        //根据layerMsg.type()信息在gParseTable中找到相应的layer层解析函数
        LayerParseFnMap::iterator v = gParseTable.find(layerMsg.type());
        if (v == gParseTable.end())
        {
            gLogError << "could not parse layer type " << layerMsg.type() << std::endl;
            ok = false;
        }
        else
        {
            //如果找到相应的layer层解析函数，则直接调用相应的解析函数对层进行解析？
            //weights变量包含了所有层的weight数据，解析层的时候可以用上
            ILayer* layer = (*v->second)(network, layerMsg, weights, mBlobNameToTensor);
            if (layer == 0)
            {
                gLogError << "error: parsing layer type " << layerMsg.type() <<
                    " index " << i << std::endl;
                ok = false;
            }
            else
            {
                layer->setName(layerMsg.name().c_str());
                mBlobNameToTensor->add(layerMsg.top(0), layer->getOutput(0));
            }
        }
    }
    //为表格中每个tensor设定name，其实就是把tensor的name设定成blobname
    mBlobNameToTensor->setTensorNames();
    return ok && weights.isOK() ? mBlobNameToTensor : 0;
}
//上面的函数用到了BlobNameToTensor的class，这个class实现了一个string到ITensor*的映射map数据结构
class BlobNameToTensor : public IBlobNameToTensor
{
public:
    virtual void add(const std::string& name, ITensor* tensor);
    virtual ITensor* find(const char* name) const;
    virtual ITensor*& operator[](const std::string& name);
    virtual void setTensorNames();
    virtual ~BlobNameToTensor();
private:
    std::map<std::string, ITensor*> mMap; //blobName到ITensor*的映射map
};
```

```protobuf
message NetParameter {
  optional string name = 1; // consider giving the network a name
  // DEPRECATED. See InputParameter. The input blobs to the network.
  repeated string input = 3;
  // DEPRECATED. See InputParameter. The shape of the input blobs.
  repeated BlobShape input_shape = 8;
  // 4D input dimensions -- deprecated.  Use "input_shape" instead.
  // If specified, for each input blob there should be four
  // values specifying the num, channels, height and width of the input blob.
  // Thus, there should be a total of (4 * #input) numbers.
  repeated int32 input_dim = 4;
  // Whether the network will force every layer to carry out backward operation.
  // If set False, then whether to carry out backward is determined
  // automatically according to the net structure and learning rates.
  optional bool force_backward = 5 [default = false];
  // The current "state" of the network, including the phase, level, and stage.
  // Some layers may be included/excluded depending on this state and the states
  // specified in the layers' include and exclude fields.
  optional NetState state = 6;
  // Print debugging information about results while running Net::Forward,
  // Net::Backward, and Net::Update.
  optional bool debug_info = 7 [default = false];
  // The layers that make up the net.  Each of their configurations, including
  // connectivity and behavior, is specified as a LayerParameter.
  repeated LayerParameter layer = 100;  // ID 100 so layers are printed last.
  // DEPRECATED: use 'layer' instead.
  repeated V1LayerParameter layers = 2;
}
message LayerParameter {
  optional string name = 1; // the layer name
  optional string type = 2; // the layer type
  repeated string bottom = 3; // the name of each bottom blob
  repeated string top = 4; // the name of each top blob
  // The train / test phase for computation.
  optional Phase phase = 10;
  // The amount of weight to assign each top blob in the objective.
  // Each layer assigns a default value, usually of either 0 or 1,
  // to each top blob.
  repeated float loss_weight = 5;
  // Specifies training parameters (multipliers on global learning constants,
  // and the name and other settings used for weight sharing).
  repeated ParamSpec param = 6;
  // The blobs containing the numeric parameters of the layer.
  repeated BlobProto blobs = 7;
  // Specifies whether to backpropagate to each bottom. If unspecified,
  // Caffe will automatically infer whether each input needs backpropagation
  // to compute parameter gradients. If set to true for some inputs,
  // backpropagation to those inputs is forced; if set false for some inputs,
  // backpropagation to those inputs is skipped.
  // The size must be either 0 or equal to the number of bottoms.
  repeated bool propagate_down = 11;
  // Rules controlling whether and when a layer is included in the network,
  // based on the current NetState.  You may specify a non-zero number of rules
  // to include OR exclude, but not both.  If no include or exclude rules are
  // specified, the layer is always included.  If the current NetState meets
  // ANY (i.e., one or more) of the specified rules, the layer is
  // included/excluded.
  repeated NetStateRule include = 8;
  repeated NetStateRule exclude = 9;
  // Parameters for data pre-processing.
  optional TransformationParameter transform_param = 100;
  // Parameters shared by loss layers.
  optional LossParameter loss_param = 101;
  // Layer type-specific parameters.
  //
  // Note: certain layers may have more than one computational engine
  // for their implementation. These layers include an Engine type and
  // engine parameter for selecting the implementation.
  // The default for the engine is set by the ENGINE switch at compile-time.
  optional AccuracyParameter accuracy_param = 102;
  optional ArgMaxParameter argmax_param = 103;
  optional BatchNormParameter batch_norm_param = 139;
  optional BiasParameter bias_param = 141;
  optional ConcatParameter concat_param = 104;
  optional ContrastiveLossParameter contrastive_loss_param = 105;
  optional ConvolutionParameter convolution_param = 106;
  optional CropParameter crop_param = 144;
  optional DataParameter data_param = 107;
  optional DropoutParameter dropout_param = 108;
  optional DummyDataParameter dummy_data_param = 109;
  optional EltwiseParameter eltwise_param = 110;
  optional ELUParameter elu_param = 140;
  optional EmbedParameter embed_param = 137;
  optional ExpParameter exp_param = 111;
  optional FlattenParameter flatten_param = 135;
  optional HDF5DataParameter hdf5_data_param = 112;
  optional HDF5OutputParameter hdf5_output_param = 113;
  optional HingeLossParameter hinge_loss_param = 114;
  optional ImageDataParameter image_data_param = 115;
  optional InfogainLossParameter infogain_loss_param = 116;
  optional InnerProductParameter inner_product_param = 117;
  optional InputParameter input_param = 143;
  optional LogParameter log_param = 134;
  optional LRNParameter lrn_param = 118;
  optional MemoryDataParameter memory_data_param = 119;
  optional MVNParameter mvn_param = 120;
  optional ParameterParameter parameter_param = 145;
  optional PoolingParameter pooling_param = 121;
  optional PowerParameter power_param = 122;
  optional PReLUParameter prelu_param = 131;
  optional PythonParameter python_param = 130;
  optional ReductionParameter reduction_param = 136;
  optional ReLUParameter relu_param = 123;
  optional ReshapeParameter reshape_param = 133;
  optional ScaleParameter scale_param = 142;
  optional SigmoidParameter sigmoid_param = 124;
  optional SoftmaxParameter softmax_param = 125;
  optional SPPParameter spp_param = 132;
  optional SliceParameter slice_param = 126;
  optional TanHParameter tanh_param = 127;
  optional ThresholdParameter threshold_param = 128;
  optional TileParameter tile_param = 138;
  optional WindowDataParameter window_data_param = 129;
}
```

​	这个函数中，有用到gParseTable这个层解析函数表，表格中存放的是从caffe模型中读取的各个层的对应解析函数表：

```c++
LayerParseFnMap::value_type gParseTableData[] =
{
        LayerParseFnMap::value_type("Convolution", parseConvolution),
        LayerParseFnMap::value_type("Pooling", parsePooling),
        LayerParseFnMap::value_type("InnerProduct", parseInnerProduct),
        LayerParseFnMap::value_type("ReLU", parseReLU),
        LayerParseFnMap::value_type("Softmax", parseSoftMax),
        LayerParseFnMap::value_type("SoftmaxWithLoss", parseSoftMax),
        LayerParseFnMap::value_type("LRN", parseLRN),
        LayerParseFnMap::value_type("Power", parsePower),
        LayerParseFnMap::value_type("Eltwise", parseEltwise),
        LayerParseFnMap::value_type("Concat", parseConcat),
        LayerParseFnMap::value_type("Deconvolution", parseDeconvolution),
        LayerParseFnMap::value_type("Sigmoid", parseSigmoid),
        LayerParseFnMap::value_type("TanH", parseTanH),
        LayerParseFnMap::value_type("BatchNorm", parseBatchNormalization),
        LayerParseFnMap::value_type("Scale", parseScale)
};
const int nelems = sizeof gParseTableData / sizeof gParseTableData[0];
LayerParseFnMap gParseTable( gParseTableData, gParseTableData + nelems);

typedef ILayer*(*LayerParseFn)(INetwork *, const dc::LayerParameter&, CaffeWeightFactory&,
                                      IBlobNameToTensor *);
```

​	可以看到，对应caffe模型中的每一种layer，都有相应的解析函数，这些解析函数的功能都类似，负责解析一个layer，然后调用network提供的API，自动构造一个network网络内存模型。整个caffe模型到network内存表示的解析比较复杂，其中用到了google开源的protobuf库，实现在CaffePaser.cpp文件当中。

### 4.3.4.代码流程分析-network内部表示到canonical_ast::Graph图表示

前一阶段主要工作是使用protobuf库，根据ditcaffe.proto文件的定义，解析compiler的输入文件*.prototxt和*.caffemodel，生成protobuf的deserialize格式数据NetParameter，然后根据NetParameter数据（包括net的层定义和weight等信息），根据每层的type调用network类的各种API构建一个network内存表示对象，如下：

```c++
class Network : public INetwork
{
public: // externally facing

    virtual ITensor* addInput(const char* name, Dims4 dimensions);

    //	virtual void markChanged(const ILayer*);
    virtual bool markInput(ITensor * tensor);
    virtual void markOutput(ITensor* tensor);

    //下面是从INetwork接口继承过来的network构造用的API函数
    virtual IConvolutionLayer *    addConvolution(ITensor* input, int numOutputs, int paddingValue,
Dims2 kernelSize, Dims2 tlPadding, Dims2 brPadding, Dims2 stride, Dims2 dilation, Weights kernelWeights, Weights biasWeights, BiasMode biasmode, int numGroups);
    virtual IFullyConnectedLayer * addFullyConnected(ITensor* input, int outputSize, Weights kernelWeights, Weights biasWeights, BiasMode biasMode);
    virtual IActivationLayer *     addActivation(ITensor* input, ActivationType type);
    virtual IPoolingLayer *        addPooling(ITensor* input, PoolingType type,Dims2 windowSize, Dims2 stride, Dims2 tlPadding, Dims2 brPadding);
    virtual ILRNLayer *            addLRN(ITensor* input, int window, float alpha, float beta, float k);
    virtual IScaleLayer *          addScale(ITensor* input, ScaleMode mode, Weights shift, Weights scale, Weights power);
    virtual IBatchNormLayer *      addBatchNorm(ITensor* input, BatchNormMode mode, Weights mean, Weights variance, float epsilon);
    virtual ISoftMaxLayer *        addSoftMax(ITensor* input);
    virtual IConcatenationLayer *  addConcatenation(ITensor * const * inputs, int numInputs);
    virtual ISliceLayer *          addSlice(ITensor* input, int numOutputs);
    virtual IDeconvolutionLayer *  addDeconvolution(ITensor* input, int numOutputs, int paddingValue,
Dims2 kernelSize, Dims2 tlPadding, Dims2 brPadding, Dims2 stride, Dims2 dilation,Weights kernelWeights, Weights biasWeights, BiasMode biasMode, int numGroups);
    virtual IElementWiseLayer *    addElementWise(ITensor* input0, ITensor* input1, ElementWiseOperation op);

    virtual int  getNumInputs() const;
    virtual int  getNumOutputs() const;
    virtual int  getNumLayers() const ;

    virtual ILayer  * getLayer(int index)  const;
    virtual ITensor * getOutput(int index) const;
    virtual ITensor * getInput(int index)  const;

    virtual void setPoolingOutputDimensionsFormula      (OutputDimensionsFormula* callback);
    virtual void setConvolutionOutputDimensionsFormula  (OutputDimensionsFormula* callback);
    virtual void setDeconvolutionOutputDimensionsFormula(OutputDimensionsFormula* callback);

    virtual OutputDimensionsFormula& getPoolingOutputDimensionsFormula()       const;
    virtual OutputDimensionsFormula& getConvolutionOutputDimensionsFormula()   const;
    virtual OutputDimensionsFormula& getDeconvolutionOutputDimensionsFormula() const;

    virtual const std::vector<ITensor *>& getInputs()  const;
    virtual const std::vector<ILayer * >& getLayers()  const;
    virtual const std::vector<ITensor *>& getOutputs() const;

    virtual NvU16 getFactoryType() const;
public: // internally facing
    Network();
    virtual ~Network();
    virtual bool serializeTo(WisdomContainerEntry *) const;
    virtual bool deserializeFrom(WisdomContainerEntry *);
    virtual bool assignSymbols(Wisdom *);

protected:
    friend class Wisdom;
    friend class NetworkFactory;
    void destroy();
private:
    std::string newLayerName() const;
    std::string newTensorName() const;
    ITensor* addTensor(const std::string & s);
    const ILayer* findLayer(const std::string& name) const;
    bool checkNames(const char* name);

    std::vector<ITensor *> mTensors;//network的tensor
    std::vector<ILayer *>  mLayers; //network的layers
    std::vector<ITensor *> mInputs; //network的inputTensor
    std::vector<ITensor *> mOutputs;//network的outputTensor

    // provides layer dimension caching. Layers can be mutated in any order and dimensions queried at any point.
    // So mutating a layer trims this, and querying always refills the cache up to the queried layer
    //	mutable std::vector<Dims3> mDimensions;

    // internal flags used by the builder that are not accessible through the API
    // int mInternalBuildFlags{ InternalBuildFlags::kENABLE_GRAPH_OPTIMIZATIONS };
    OutputDimensionsFormula* mConvDims, *mDeconvDims, *mPoolDims;
};
```

可以看到，除了一系列各种type的layer添加API函数以外，还有就是下列几个private数据变量：

```c++
    std::vector<ITensor *> mTensors;//network的tensor
    std::vector<ILayer *>  mLayers; //network的layers
    std::vector<ITensor *> mInputs; //network的inputTensor
    std::vector<ITensor *> mOutputs;//network的outputTensor
```

可以看到，这几个变量只是简单的记录了network的layer、tensor、input和output等信息，并没有图的节点边缘以及连接先后关系等概念。下一阶段应该是根据network数据结构构建CAG图的工作。

​	从network内部表示到canonical_ast::Graph图表示，这一部分功能主要是由canonical_ast::generateGraph()这个函数完成的，输入是一个network对象，只有layer和tensor概念，输出就变成一个canonical_ast::Graph图对象，有了node和edge的概念，下面分析canonical_ast::generateGraph()函数代码：

```c++
canonical_ast::Graph *canonical_ast::generateGraph(Network *network)
{
    vector<canonical_ast::Edge *> input_edges;//graph的输入edges
    vector<canonical_ast::Edge *> output_edges;//graph的输出edges
	
    //network中的layer和Graph中的node的映射map
    map<canonical_ast::Node *, Layer *, Graph::nodeCompareFn>  node_layer;
    map<canonical_ast::Node *, Layer *, Graph::nodeCompareFn>::iterator lni;

    map<Tensor *, canonical_ast::Edge *>  tensor_edge;//nework中tensor和Graph中Edge的映射MAP
    map<Tensor *, Tensor *>  nw_tensor_to_can_tensor;//network中的tensor和Graph中的tensor的映射MAP
    map<Tensor *, canonical_ast::Edge *>::iterator tei;

    Graph *graph = new Graph();//新建一个canonical_ast::Graph
	
    //下面这个循环把network中的inputTensor汇总成一个数组，并把Graph中input_edges数组大小设定成
    //network中的inputTensor的数组大小
    vector<Tensor *> network_inputs;
    for (int ni = 0; ni < network->getNumInputs(); ++ni)
    {
        network_inputs.push_back(TensorFactory::priv(network->getInput(ni)));
    }
    input_edges.resize(network_inputs.size());

    //下面这个循环把network中的outputTensor汇总成一个数组，并把Graph中outputedges数组大小设定成
    //network中的outputTensor的数组大小
    vector<Tensor *> network_outputs;
    for (int ni = 0; ni < network->getNumOutputs(); ++ni)
    {
        network_outputs.push_back(TensorFactory::priv(network->getOutput(ni)));
    }
    output_edges.resize(network_outputs.size());

    //下面这个循环迭代network中的layer序列，根据每个layer的信息分别建立Graph中的相应的Node
    for (int li = 0; li < network->getNumLayers(); li++)
    {
        ILayer *ilayer = network->getLayer(li);
        Layer *layer = LayerFactory::priv(ilayer);
        if ( !(ilayer && layer) )
        {
            gLogError << __func__ << "encountered null layer at network layer index=" << li << endl;
            continue;
        }
		
        //根据network中的layer，建立相应的Node
        canonical_ast::Node *can_node = newCanonicalNode(layer);
        if ( !can_node )
        {
            delete graph; // blow up
            graph = 0;
            goto done;
        }
        can_node->setGraph(graph);//指定node的container是graph
        graph->insertNode(can_node);//把node加入graph的node序列当中

        can_node->setId(graph->nextNodeId());//设定node的id序号，string类型，例如n-0,n-1等递增的
        can_node->setName(layer->getName());//node的name设定成network中对应layer的name
        
		//在node_layer映射map中添加一项，记录graph中node和network中对应layer的对应关系
        node_layer[can_node] = layer;
    }

    //现在，所有network中的layer都在graph中建立了相应的node，并且这个对应关系也记录在了node_layer的MAP中
    //下面循环迭代这个MAP中的每一项
    for (lni = node_layer.begin(); lni != node_layer.end(); ++lni)
    {
        canonical_ast::Node *node = lni->first;
        Layer *l = lni->second;

        size_t input_tensors = 0, output_tensors = 0, aux_input_tensors = 0;
        vector<Tensor *> io_tensors, aux_tensors;
        NVDLA_UNUSED(aux_input_tensors);
		//针对network中当前迭代的这个layer，找出其全部inputTensors并加入io_tensors列表
        for(int ii = 0, II = l->getNumInputs(); ii < II; ++ii)
        {
            Tensor *tensor = TensorFactory::priv(l->getInput(ii));
            if ( !tensor )
            {
                gLogError << __func__ << " 3.<null>.i." << ii << endl;
                continue;
            }
            io_tensors.push_back(tensor);
            input_tensors++;
        }
        //针对network中当前迭代的这个layer，找出其全部outputTensors并加入io_tensors列表
        for(int oo = 0, OO = l->getNumOutputs(); oo < OO; ++oo)
        {
            Tensor *tensor = TensorFactory::priv(l->getOutput(oo));
            if ( ! tensor )
            {
                gLogError << __func__ << " 3.<null>.o." << oo << endl;
                continue;
            }
            io_tensors.push_back(tensor);
            output_tensors++;
        }
		//针对当前layer，迭代刚刚找到的全部iotensor的列表
        for(size_t io = 0, IO = io_tensors.size(); io < IO; ++io)
        {
            Tensor *nw_tensor = io_tensors[io];
            bool is_input = io < input_tensors;//根据当前tensor在列表中的位置判断是input还是output
            //edge_side是个enum值，input=SECOND，output=FIRST
            ast::EdgeSide edge_side(is_input ? ast::EdgeSideEnum::SECOND:ast::EdgeSideEnum::FIRST);
            //edge_dir是个enum值，有单向双向和无方向三种，这里统一设定为单向
            ast::EdgeDirection edge_dir(ast::EdgeDirectionEnum::DIRECTED);
		   //在tensor_edge映射MAP中查找当前tensor的对应项
            map<Tensor *, canonical_ast::Edge *>::iterator f = tensor_edge.find(nw_tensor);
            canonical_ast::Edge *can_edge = 0;//graph中的edge
            Tensor* can_tensor = 0;//graph中的tensor
            if ( f == tensor_edge.end() )//如果没有在MAP中找到对应项
            {
                can_edge = new canonical_ast::Edge();//新建一个graph中的edge
                can_edge->setGraph(graph);//把新建的edge的container设定为graph

                can_tensor = nw_tensor->clone();//把network中的tensor复制到一个新的变量can_tensor
                can_tensor->setNetwork(NULL);   //由于这个新的tensor变量将加入graph所以其network指针清空，不在指向原来的network(这里是复制一份tensor，network中原来的tensor还在)
                can_tensor->setTensorType(TensorType::kIO);//graph中的tensor设定为IO类型
                can_edge->setId(graph->nextEdgeId());//graph中edge的Id设定为string，e-0,e-1,e-2等
                can_edge->setOriginalTensor(can_tensor);//graph中的edge的原始tensor设定为can_tensor，注意，这里的OriginalTensor指向的是从network中复制clone过来的一个副本，并不在network中，可以看出这里的包含关系，graph-->can_edge-->can_tensor
                graph->insertEdge(can_edge);//把根据network中1个layer的iotensor新建的edge加入graph列表
			  
                tensor_edge[nw_tensor] = can_edge;//tensor_edge映射MAP加入nw中tensor到graph中edge映射
                //nw_tensor_to_can_tensor映射MAP加入nw中tensor到graph中edge的tensor映射
                nw_tensor_to_can_tensor[nw_tensor] = can_tensor;
            } else {
                can_edge = f->second;
            }
            //把当前新建的edge加入到node的edge_side侧列表当中
            graph->appendNodeToEdge(can_edge, edge_side, node);

            // if this is an input node it could be one of the network inputs.
            // if so keep track of it.
            if ( is_input )
            {
                //迭代整个network的inputTensors列表
                for ( size_t iti = 0; iti < network_inputs.size(); iti++)
                {
                    //如果当前node对应的这个inputTensor在整个network的inputTensors列表当中
                    if ( nw_tensor == network_inputs[iti] )
                    {
                        input_edges[iti] = can_edge;//把当前edge加入graph的input_edges列表当中
                        can_tensor = nw_tensor_to_can_tensor[nw_tensor];
                        //设定当前tensor属性为INPUT
                        can_tensor->setTensorType(TensorType::kNW_INPUT);
                        break;
                    }
                }
                node->markInputEdge(can_edge);//告诉当前node，你的这个edge是一个网络inputedge
            }
            else
            {
                //这部分解释参考上面
                for ( size_t oti = 0; oti < network_outputs.size(); oti++)
                {
                    if ( nw_tensor == network_outputs[oti] )
                    {
                        output_edges[oti] = can_edge;
                        can_tensor = nw_tensor_to_can_tensor[nw_tensor];
                        can_tensor->setTensorType(TensorType::kNW_OUTPUT);
                        break;
                    }
                }
                node->markOutputEdge(can_edge);
            }
        }
    }

    if ( input_edges.size() )
    {
        graph->setInputEdges(input_edges);//设定整个graph的inputedges队列为input_edges
    }
    if ( output_edges.size() )
    {
        graph->setOutputEdges(output_edges);//设定整个graph的outputedges队列为input_edges
    }

    graph->scoredOrdering()->generate();//graph计分牌生成，这部分比较复杂，单独讲
    graph->markClean();//清除graph的m_dirty脏标志，所有对graph的更改都要设定m_dirty为true

done:
    return graph;//把按照network生成的graph作为返回值返回
}
```

### 4.3.5.代码流程分析-canonical_ast::Graph到engine_ast::Graph图表示

### 4.3.6.代码流程分析-EngineAST中间IR变换与优化PASS

### 4.3.7.代码流程分析-EngineAST到后端代码Emit（代码生成）
