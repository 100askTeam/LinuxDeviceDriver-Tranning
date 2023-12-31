# V4L2应用程序开发 #

参考资料：

* mjpg-streamer：https://github.com/jacksonliam/mjpg-streamer
* video2lcd：这是百问网编写的APP，它可以在LCD上直接显示摄像头的图像
* 这2个源码都放在GIT仓库里：
  ![image-20230617173258521](pic/04_demo.png)



## 1. 

To be able to manipulate the physical properties of a video function, its functionality must be
divided into addressable entities. The following two generic entities are identified:

* Units：Selector Unit  、Processing Unit  、Encoding Unit  、Extension Unit  
*  Terminals ：Input Terminal (IT)   、Output Terminal (OT)  、Camera Terminal (CT)  
  * A Camera Terminal is always represented as an
    Input Terminal with a single output pin. It provides support for the following features  
    * Scanning Mode (Progressive or Interlaced)
    *  Auto-Exposure Mode
    *  Auto-Exposure Priority
    *  Exposure Time
    *  Focus
    *  Auto-Focus
    *  Simple Focus
    *  Iris
    *  Zoom  
    * Pan
    *  Roll
    *  Tilt
    *  Digital Windowing
    *  Region of Interest  

Controls have attributes, which might include:  

* Current setting
*  Minimum setting
*  Maximum setting
*  Resolution
*  Size
*  Default  

Processing Unit  

* User Controls  
  * Brightness
  *  Hue
  *  Saturation
  *  Sharpness
  *  Gamma
  *  Digital Multiplier (Zoom)  
* Auto Controls  
  * White Balance Temperature
  *  White Balance Component
  *  Backlight Compensation  
  * Contrast  
* Other  
  * Gain
  *  Power Line Frequency
  *  Analog Video Standard
  *  Analog Video Lock Status  



Extension Unit  





对于每一个entity(IT,PU,SU,OT等)

* IT: input terminal
* OT：output terminal
* VC: video control
* 

video function：含有几个interface，每个video function含有一个VC(video control)、多个VS(video streaming)，它们被称为VIC。



参考资料：

https://www.php1.cn/detail/CSS_YangShiGuiZe_218a96fe.html

