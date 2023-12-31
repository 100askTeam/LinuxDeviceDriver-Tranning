# 站点简述(Introduction)

!!! abstract "目标"

    嵌入式Linux全新系列教程之驱动大全：基于百问网IMX6ULL-PRO开发板，STM32MP157-PRO开发板录制，深入讲解Linux内核驱动框架组成，涉及通信协议原理，手把手从零编写代码，手绘+代码现场编写刨析方式 带您深入理解各个部分。


## 如何阅读此站？

{==

感兴趣可以加入QQ群 562614605了解，建议使用PC端浏览器访问，分辨率1080p为最佳阅读。
IMX6ULL-PRO开发板商品介绍：https://item.taobao.com/item.htm?&id=610613585935
STM32MP157-PRO开发板商品介绍：https://item.taobao.com/item.htm?&id=623233533961
==}

### 嵌入式开发流程简述
  在进入嵌入式Linux开发之前，我们需要先了解一下嵌入式系统的开发框架以及流程，如下图所示，左侧为我们常用的PC电脑主机，右侧为嵌入式系统，因为嵌入式芯片的性能内存存储条件等原因 无法直接在嵌入式芯片上进行程序开发，此时我们就需要借助性能强大的PC主机，一般为X86电脑，通过交叉编译的方式生产 嵌入式芯片可以运行的程序，在这里，我们一般叫PC主机为Host端，对于我们的嵌入式开发板，我们一般成为Target端。我们需要在Host端使用针对于Target端芯片架构生成的工具链进行交叉编译，最后将输出的可执行文件存放至Target(目标开发板)内运行。
![eLinuxHostAndTarget](https://cdn.staticaly.com/gh/DongshanPI/LinuxCodeLibrary-Photos@master/eLinuxHostAndTarget.jpg)

  通过上图可以看出，实际的嵌入式开发是通过两部分组成的，那么对于嵌入式Linux设备开发也是如此。通过Host端的交叉编译工具链编译 Target(目标开发板)的程序，然后放在Target开发板上运行。
![cross-gcc-sandwich](https://cdn.staticaly.com/gh/DongshanPI/LinuxCodeLibrary-Photos@master/cross-gcc-sandwich.png)  

  既然Target(目标开发板)上运行的程序是通过Host(PC主机)端交叉编译工具链生成，那么交叉编译工具链又是从哪里而来？这里就涉及到了一个BuildSystem(构建系统)的概念，所有的程序都不可能是直接凭空出现，而是通过编程开发，最后再编译得出。下面这张图演示了，交叉编译工具链的生成以及如何使用交叉编译工具链编译生成一个可以在Target(目标开发板)上运行的c++程序。这里主要告诉大家，所有的程序都是从源码编译得出，不管是交叉编译工具链还是 Target上运行的程序。
![gcc-cross-compiler](https://cdn.staticaly.com/gh/DongshanPI/LinuxCodeLibrary-Photos@master/gcc-cross-compiler.png)

* 如果看了上述的介绍还是不太理解 嵌入式的开发流程框架可以看我们之前专门录制的一套基于buildroot开发的嵌入式基础知识科普类视频教程。
<iframe width="800px" height="600px" src="//player.bilibili.com/player.html?aid=897646032&bvid=BV1VN4y137Tf&cid=753046575&page=9" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>


### 代码库使用步骤
!!! note
    详细的各个部分开发步骤请查看 左侧导航  [LinuxC基础](/01-LinuxCprogrammers/)  [Linux组件开发](/02-Components/) [Linux设备驱动开发](/03-DeviceDriver/) [Linux系统开发](04-System/) 页面，在页面内会有详细的各部分开发流程介绍。

``` mermaid
graph LR
A[0.准备硬件]-->B[1.获取Ubuntu虚拟机系统]-->2.配置Host开发环境-->3.获取Target工程示例-->4.Host端编译开发-->5.上传至Target开发板运行
```


## 如何参与编辑？

* 我们所有的文档使用makrdown进行编写，存放在github仓库https://github.com/100askTeam/LinuxDeviceDriver-Tranning 内，可以直接点击每个页面右上角的 🖊 箭头直接编辑修改。
* 对于代码示例等工程，全部都根据不同的使用场景存放在不同的git仓库内。

> 愿景：我们正在不断地支持更多的硬件模块，代码示例，加入到这个站点内，如果您感兴趣，欢迎加入。

* 讨论交流：https://forums.100ask.net/c/elinuxdev/23


## 您将获得什么？
快速实现您需要实现的功能，我们提供专门的开发板硬件，配套的模块硬件，设备驱动 系统源码SDK工程源码，常用组件开发示例，以及LinuxC基础，让大家可以在最短的时间内实现自己需要的功能。


## 关于开源协议
  此页面使用了开源的Mkdoc文档框架，文档站点托管在GitHub上，每个页面都会有编辑按钮，大家可以一起参与编辑或者提问改进此文档,如果您转发此文章，请注明出处，谢谢！
* 图片引用: https://preshing.com/20141119/how-to-build-a-gcc-cross-compiler/