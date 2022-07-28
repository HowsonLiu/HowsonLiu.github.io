---
title: 音视频编码原理
date: 2020-01-31 11:55:45
categories: 音视频
tags:
    - 音视频
    - 原理
description: 简单记录学习音视频时看到的一些原理与编码方式
---
# 视频播放的流程
![](1.jpg)  
1. **解协议**  
    解协议的作用，就是将流媒体协议的数据，解析为标准的相应的封装格式数据。音视频在网络上传播的时候，常常采用各种流媒体协议，例如HTTP，RTMP，或是MMS等等。这些协议在传输音视频数据的同时，也会传输一些信令数据。这些信令数据包括对播放的控制（播放，暂停，停止），或者对网络状态的描述等。解协议的过程中会去除掉信令数据而只保留音视频数据。
2. **解封装**  
    解封装的作用，是将输入的封装格式的数据，分离成为音频流压缩编码数据以及视频流压缩编码数据。封装格式的种类很多，比如MP4，MKV，RMVB，FLV等等，它的作用就是将已经压缩编码的视频数据和音频数据按照一定的格式放到一起。
3. **解码**  
    解码的作用，就是将视频/音频压缩编码数据，解码成为非压缩的视频/音频原始数据。音频的压缩编码标准包含AAC，MP3，AC-3等等，视频的压缩编码标准则包含H.264，MPEG2，VC-1等等。解码是整个系统中最重要也是最复杂的一个环节。通过解码，压缩编码的视频数据输出成为非压缩的颜色数据，例如YUV420P，RGB等等；压缩编码的音频数据输出成为非压缩的音频抽样数据，例如PCM数据。
4. **音视频同步**  
    视音频同步的作用，就是根据解封装模块处理过程中获取到的参数信息，同步解码出来的视频和音频数据，并将视频音频数据送至系统的显卡和声卡播放出来。

# 视频编码基本原理
## 视频信号的冗余信息
总所周知，视频图像是由一连串的图片组成的，图片之间差距不大。当切换图片的速度超过每秒24帧时，根据[视觉暂留原理和似动现象](https://blog.csdn.net/charleslei/article/details/53248561)，人眼无法辨别单张图片，会认为这是平滑连续的视觉效果。
>以记录数字视频的YUV分量格式为例，YUV分别代表亮度与两个色差信号。例如对于现有的PAL制电视系统，其亮度信号采样频率为13.5MHz；色度信号的频带通常为亮度信号的一半或更少，为6.75MHz或3.375MHz。以4：2：2的采样频率为例，Y信号采用13.5MHz，色度信号U和V采用6.75MHz采样，采样信号以8bit量化，则可以计算出数字视频的码率为：
13.5\*8 + 6.75\*8 + 6.75\*8= 216Mbit/s。
如此大的数据量如果直接进行存储或传输将会遇到很大困难，因此必须采用压缩技术以减少码率。

数字化的视频信号能进行压缩主要依据两个条件：
- **数据冗余**  
比如空间冗余，时间冗余，结构冗余，信息熵冗余等。即图像的各像素之间存在很强的相关性，消除这些冗余并不会导致信息丢失，属于无损压缩
- **视觉冗余**  
人眼的一些特性比如亮度辨别阈值，视觉阈值，对亮度和色度的敏感度不同，使得在编码的时候引入适量的误差，也不会被察觉出来。可以利用人眼的视觉特性，以一定的客观失真换取数据压缩。这种压缩属于有损压缩。

## 压缩编码的种类
- **帧内编码**  
    帧内编码是空间域编码，利用图像空间性冗余度进行图像压缩，处理的是一幅独立的图像，不会跨越多幅图像。例如JPEG标准。
- **帧间编码**  
    帧间编码是时间域编码，是利用一组连续图像间的时间性冗余度进行图像压缩。如果某帧图像可被解码器使用，那么解码器只须利用两帧图像的差异即可得到下一帧图像。例如MPEG标准。

## 压缩编码的方法
- **变换编码**  
    变换编码的作用是将空间域描述的图像信号变换到频率域，然后对变换后的系数进行编码处理。一般来说，图像在空间上具有较强的相关性，变换到频率域可以实现去相关和能量集中。常用的正交变换有离散傅里叶变换，离散余弦变换等等。数字视频压缩过程中应用广泛的是离散余弦变换。**属于帧内编码。**
- **熵编码**  
    熵编码多用可变字长编码（VLC，Variable Length Coding）实现。其基本原理是对信源中出现概率大的符号赋予短码，对于出现概率小的符号赋予长码，从而在统计上获得较短的平均码长。可变字长编码通常有霍夫曼编码、算术编码、游程编码等。其中游程编码是一种十分简单的压缩方法，它的压缩效率不高，但编码、解码速度快，仍被得到广泛的应用，特别在变换编码之后使用游程编码，有很好的效果。**属于帧内编码。**
- **运动估计（Motion Estimation）和运动补偿（Motion Compensation）**  
    运动估计指的是从视频序列中抽取运动信息的一整套技术。运动补偿指的是通过先前的局部图像来预测、补偿当前的局部图像。**属于帧间编码。**
- **混合编码**  
    上面介绍了视频压缩编码过程中的几个重要的方法。在实际应用中这几个方法不是分离的，通常将它们结合起来使用以达到最好的压缩效果。例如MPEG1，MPEG2，H.264等标准。

## 细看运动估计与运动补偿
### 理论基础
一段时间内图像的统计结果表明，在相邻几幅图像画面中，一般有差别的像素只有10%以内的点，亮度差值变化不超过2%，而色度差值的变化只有1%以内。所以对于一段变化不大的图像画面，我们可以先编码出一个完整的图像帧A，随后的B帧不编码全部图像，只写入与A帧的差别，这样B帧的大小就只有完整帧的1/10或更小！
### 编码解码流程
运动估计在通过运动估计算法得到运动矢量之后，会根据运动矢量计算出预测帧，接着再计算预测帧与实际帧的差值Δf，完成编码流程。在解码流程中，运动补偿也会根据运动矢量计算出预测帧，最后加上差值Δf，得到实际帧，完成解码流程。
### 帧种类
- **I帧**(Intra-coded picture)：帧内编码帧，又叫关键帧。包含一幅完整的图像信息，属于帧内编码图像，不含运动矢量，在解码时不需要参考其他帧图像。
- **IDR帧**(Instantaneous Decoding Refresh picture)：即时解码刷新帧。当解码器解码到ID R帧的时候，会将前后向参考帧列表清空，将已解码的数据全部输出或者丢弃，然后开始一次全新的解码序列。
- **P帧**(Predictive-coded picture)：预测编码图像帧，是帧间编码帧，利用前面的I帧或者P帧进行预测编码
- **B帧**(Bi-directionally predicted picture)：双向预测编码图像帧，是帧间编码帧，利用之前或者之后的I帧或者P帧进行双向预测编码。B帧不可以作为参考帧。

### GOP(Group Of Pictures)
GOP是一组连续的图像，有一个I帧和多个B/P帧组成，是编解码器存取的基本单位。可以分为：
- **闭合式GOP**：闭合式GOP只需要参考本GOP内的图像即可，不需参考前后GOP的数据。这种模式决定了，闭合式GOP的显示顺序总是以I帧开始以P帧结束
- **开放式GOP**：开放式GOP中的B帧解码时可能要用到其前一个GOP或后一个GOP的某些帧。码流里面包含B帧的时候才会出现开放式GOP
    ![](https://leichn.github.io/img/avideo_basics/gop_mode.jpg)

### DTS和PTS
- **DTS**(Decoding Time Stamp)：解码时间戳，表示packet的解码时间
- **PTS**(Presentation Time Stamp)：显示时间戳，表示packet解码后数据的显示时间
    ![](https://leichn.github.io/img/avideo_basics/decode_order.jpg)

# 音频编码基本原理
## 音频信号的冗余信息
> 数字音频信号如果不加压缩地直接进行传送，将会占用极大的带宽。例如，一套双声道数字音频若取样频率为44.1KHz，每样值按16bit量化，则其码率为：
2\*44.1kHz\*16bit=1.411Mbit/s
如此大的带宽将给信号的传输和处理都带来许多困难，因此必须采取音频压缩技术对音频数据进行处理，才能有效地传输音频数据。

数字音频压缩编码是通过去除音频信号的冗余成分来实现的。其中冗余成分包括：
- 人耳听觉范围外的音频信号
- 根据人耳听觉的生理和心理声学现象，在某些情况会对某些声音进行屏蔽
    - 频谱掩蔽效应
    - 时域掩蔽效应

## 频谱掩蔽效应
![](https://img-blog.csdn.net/20140602173751953?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGVpeGlhb2h1YTEwMjA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

一个频率的声音能量小于某个阈值之后，人耳就会听不到，这个阈值称为最小可闻阈。当有另外能量较大的声音出现的时候，该声音频率附近的阈值会提高很多，即所谓的掩蔽效应。
## 时域掩蔽效应
![](https://img-blog.csdn.net/20140602173759515?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGVpeGlhb2h1YTEwMjA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

当强音信号和弱音信号同时出现时，还存在时域掩蔽效应。即两者发生时间很接近的时候，也会发生掩蔽效应。时域掩蔽过程曲线如图所示，分为前掩蔽、同时掩蔽和后掩蔽三部分。
# YUV
```
Y Y Y Y Y Y      Y Y Y Y Y Y      Y Y Y Y Y Y      Y Y Y Y Y Y
Y Y Y Y Y Y      Y Y Y Y Y Y      Y Y Y Y Y Y      Y Y Y Y Y Y
Y Y Y Y Y Y      Y Y Y Y Y Y      Y Y Y Y Y Y      Y Y Y Y Y Y
Y Y Y Y Y Y      Y Y Y Y Y Y      Y Y Y Y Y Y      Y Y Y Y Y Y
U U U U U U      V V V V V V      U V U V U V      V U V U V U
V V V V V V      U U U U U U      U V U V U V      V U V U V U
 - I420 -          - YV12 -         - NV12 -         - NV21 -

```