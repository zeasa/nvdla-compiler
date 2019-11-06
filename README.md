# <font color="green" size = "8px" face="courier new"><center>**nvdla-compiler-learning**</center></font> #
# <font color="blue">1.安装准备与一些有用的资源</font> #
## 1.1.github地址 ##
	https://github.com/nvdla
## 1.2.编译和安装 ##
### 1.2.1.参考信息 ###
	https://blog.csdn.net/zhajio/article/details/84784336 
	https://blog.csdn.net/hywCogost/article/details/82114529
### 1.2.2.问题与解决 ###
	
## 1.3.软硬件分析文章参考 ##
	https://github.com/JunningWu/Learning-NVDLA-Notes
# <font color="blue">2.源代码结构</font> #
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

# <font color="blue">3.模型编译与DLA仿真运行</font> #
# <font color="blue">4.深入源码</font> #
## 4.1.总体结构 ##
## 4.2.runtime部分 ##
## 4.3.compiler部分 ##