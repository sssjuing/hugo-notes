---
title: 前言
weight: 1
---

### 1. WebGL 简介

OpenGL 的 API 设计以内部的全局状态对象为中心，它最大限度地减少了在任何调用中需要传入和传出到 GPU 的数据量。内部状态对象基本上是一个指针的集合。你的 API 调用可以影响状态对象所指向的对象，但也可以影响状态对象本身。因此，API 调用的顺序是非常重要的。我总觉得这使得抽象和构建库变得很困难。你必须非常细致地理清所有可能干扰你进行 API 调用的指针和状态项，同时还要将指针和值恢复到之前的值，这样你的抽象才能正确生成。

![A visualization of WebGL’s internal, global state object Taken from WebGL Fundamentals](https://surma.dev/assets/internalstate.c00c7a0f.png)

## 2. WebGL 与 WebGPU

两种 API 最大的区别在于 WebGL 是有状态的，因此 WebGL 有很多全局状态，比如当前绑定哪些纹理，当前绑定哪些缓冲区，当前程序是什么，混合、深度和模具设置是什么(what the blending, depth, and stencil settings are)。相比之下，WebGPU 几乎没有全局状态，它的渲染流水线和渲染通道包含了 WebGL 中的绝大部分全局状态。创建好的流水线是不可变的，如果想改变之前的设置只能新建流水线。渲染通道有一些可修改的状态，但这些状态都局限在其自身。

另一个区别是 WebGPU 相比 WebGL 是一种更低层级的封装。WebGL 中通常是通过名称进行关联，比如顶点着色器输出一个 `varying vec2 v_texcoord`  或  `out vec2 v_texcoord`，随后在片元着色器中需要使用同样的名称才能获取到。而在 WebGPU 中，这些是通过索引或者字节偏移量连接的。使用者没有任何 API 来控制缓冲区的内容，所有这些都需要使用者自己计算字节偏移量，并在 Javascript 和 WGSL 中同步这些信息。

WebGL 负责管理 canvas 的状态，用户只需要设置 canvas 的宽高。而 WebGPU 中用户需要自己处理诸如深度缓冲区、抗锯齿、等工作，正因如此，WebGPU 可以在多个 canvas 中同步渲染，而 WebGL 只能渲染一个 canvas。

WebGPU 不生成 Mipmap，而 WebGL 中通过调用 `gl.generateMipmap` 可以由 level 0 的 Mipmap 生成所有其他等级的 Mipmap。WebGPU 中只能使用者自己生成。

在 WebGPU 中，缓冲区和纹理一旦创建后就不能修改大小，而 WebGL 中可以修改。因此在 We 把 GL 中一种常见的方式是在一开始创建 1x1 像素的纹理作为占位符，以便立即开始渲染，同时异步加载图片，等图片加载完成后更新之前的纹理。但在 WebGPU 中，纹理和缓冲区的大小、用途、格式都是不可变的，只有其中的内容可以修改。

由于 WebGPU 是在创建流水线时指定 primitive type, cull mode, and depth settings，所以如果想在一次渲染中同时对这些设置不同的值，需要创建多个流水线进行绘制。

## 2. GPU 构造

## 3. GPU 绘制过程
