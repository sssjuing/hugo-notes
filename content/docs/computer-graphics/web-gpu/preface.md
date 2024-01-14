---
title: 前言
weight: 1
---

### 1. WebGL

OpenGL 的 API 设计以内部的全局状态对象为中心，它最大限度地减少了在任何调用中需要传入和传出到 GPU 的数据量。内部状态对象基本上是一个指针的集合。你的 API 调用可以影响状态对象所指向的对象，但也可以影响状态对象本身。因此，API 调用的顺序是非常重要的。我总觉得这使得抽象和构建库变得很困难。你必须非常细致地理清所有可能干扰你进行 API 调用的指针和状态项，同时还要将指针和值恢复到之前的值，这样你的抽象才能正确生成。

![A visualization of WebGL’s internal, global state object Taken from WebGL Fundamentals](https://surma.dev/assets/internalstate.c00c7a0f.png)

## 2. GPU构造

## 3. GPU绘制过程
