+++
title = "Unity中的音频分析算法：介绍"
date = "2021-07-01 18:10:00 +0800"
tags = ["Game Design", "technology"]
categories = ["technology"]
slug = "beat_tracking_in_unity_introduction"
indent = false
dropCap = false
+++

![FT.jpg](FT.png)

> 音乐游戏中大量的节拍需要大量的人力和时间成本，那么为什么不用代码分析节奏然后自动生成呢？Audiosurf的的创造者Dylan Fitterer开创了使用代码生成音游谱面的先河，并且取得了不错的成果。节拍映射成为了音乐游戏发展的一个重要方向。


## 本系列文章

[**Unity中的音频分析算法：介绍**](/posts/tech/beat_tracking_in_unity_introduction/)

[**Unity中的音频分析算法：使用 Unity API 进行实时音频分析**](/posts/tech/audio-analysis-algorithms-in-unity-real-time-audio-analysis/)

[**Unity中的音频分析算法：预处理音频分析**](/posts/tech/audio-analysis-algorithms-in-unity-preprocessing-audio-analysis)

<!-- [演示视频链接：https://www.bilibili.com/video/BV1zY4y1h7DL/](https://www.bilibili.com/video/BV1zY4y1h7DL/) -->

## 前言

随着众多成功的节奏游戏上市, 节奏游戏已成为游戏开发中的一大热门领域. 雷亚公司的Deemo以及Cytus和Cytus2在国内音游占有绝对优势, 还有在全球市场上占有绝对份额的老牌音游OSU!. 音乐游戏中大量的节拍需要大量的人力和时间成本，那么为什么不用代码分析节奏然后自动生成呢？Audiosurf的的创造者Dylan Fitterer开创了使用代码生成音游谱面的先河，并且取得了不错的成果。节拍映射成为了音乐游戏发展的一个重要方向。

那使用代码进行自动曲谱生成的音乐游戏有很多吗？其实答案是很少。事实上，市面上大部分优秀的音游作品基本上都是Mapper们手工录入的，即便是代码处理音频的音游存在，基本上也都是拿到了音乐本身的音轨信息（如MIDI文件）。代码生成的曲谱识别率是一个瓶颈问题，生成的节拍Note对应的也不会十分完美，对于音游来说，糟糕的交互反馈意味着糟糕的游戏质量。 

不过，我还是希望实现一个音频处理算法，作为我音游开发道路上的一次尝试，我希望能给相同道路上的开发者提供一个思路。我是一个音乐爱好者，但对音乐创作一窍不通，我找到了一些音频分析算法，他们能够提供一些新思路，以便读者们用它来制作音乐游戏或者音频可视化程序，在接下来的几篇博客中，我将分享他们的思路，并且附带具体的实现方法。

在进行音频分析时，定义什么是”节拍“很重要，它可以是弹下的钢琴键、敲击鼓、弹拨吉他的音符，也可以是一个人声的一个波动。在特定的时间点，他们会同时发生，并且互相融合，我们主要的工作就是使用算法将他们分开，这样我们就得到了音频对应的节拍，并且很容易地对应他们生成节拍谱面。  

这个系列的目标是实现一个可用并且可以拓展的预处理音频的解决方案，不过在此之前，我们将会先从实时处理入手，Unity引擎内置的一些API能够比较容易地实现实时分析，而预处理需要额外处理一些更加复杂的内容。总之，我将使用Unity3D引擎实现音频的预处理分析以及实时处理分析的算法。我是用的版本为Unity 2020.3.12f1c1，我觉得他应该也能向下支持一些旧的版本吧...我之后会将项目源码放在Github仓库里面。

准备好了吗？让我们开始吧

下一篇文章：[**Unity中的音频分析算法：使用 Unity API 进行实时音频分析**](/posts/tech/audio-analysis-algorithms-in-unity-real-time-audio-analysis/)


