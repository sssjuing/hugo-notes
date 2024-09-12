---
title: 数据传递
weight: 3
---

## 1. 缓冲区

### 1.1 概述

创建缓冲区的代码如下所示：

```ts
const vertexBuffer = device.createBuffer({
  label: "Cell vertices", // 缓冲区标签, 可选, 在报错时方便定位问题
  size: vertices.byteLength, // 缓冲区大小(字节)
  // vertices是JS的TypedArray类型,
  // 其byteLength方法可获取其字节长度
  usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST, // 指示缓冲区的用法
});
```

其中不同类型的缓冲区主要由 usage 字段标识。一般常用的有顶点缓冲区、uniform 缓冲区、存储缓冲区和索引缓冲区，分别对应于 `GPUBufferUsage.VERTEX`、`GPUBufferUsage.UNIFORM`、`GPUBufferUsage.STORAGE` 和 `GPUBufferUsage.INDEX`。

> `GPUBufferUsage` 有以下属性：
>
> - COPY_SRC：可以作为  `copyBufferToBuffer()`  调用的复制源
> - COPY_DST: 指示可以将数据复制到其中
> - VERTEX: 指定将该缓冲区用于顶点数据, 可传入 `setVertexBuffer()`
> - INDEX：Buffer 可用于 index buffer，可传入  `setIndexBuffer()`
> - INDIRECT：
> - MAP_READ：暂存缓冲区（可用于从显存中获取计算结果到 JS，一般和 COPY_SRC 一起使用）
> - MAP_WRITE：
> - QUERY_RESOLVE：
> - STORAGE：存储缓冲区（一种通用缓冲区，比 Uniform 更灵活）
> - UNIFORM：Uniform 缓冲区（更高读写性能的缓冲区）

### 1.2 内存布局

[WebGPU 数据内存布局](https://webgpufundamentals.org/webgpu/lessons/zh_cn/webgpu-memory-layout.html)

在 WebGPU 中，您提供的几乎所有数据都需要在内存中进行布局，以便与着色器中的定义相匹配。在 WGSL 中，当您编写着色器时，通常会定义 struct 结构体。除了给每个属性命名外，您还必须给它定义一个类型。此外，在提供数据时，您还需要计算结构体中的特定成员将出现在缓冲区的哪个位置，此即为**内存布局**。在[WGSL](https://webgpufundamentals.org/webgpu/lessons/webgpu-wgsl.html) v1 中, 有 4 中基本类型：

- `f32` (32 位浮点数，4 字节)
- `i32` (32 位整数，4 字节)
- `u32` (32 位无符号整数，4 字节)
- `f16` (16 位浮点数，2 字节)

### 1.3 uniform buffer

除了顶点缓冲区，另一种向 GPU 传递数据的方式是使用 uniform 缓冲区。与每次调用顶点着色器会将顶点缓冲区的不同的值传入不同，uniform 在每次调用时传入值都是一样的。一般适合传递几何图形（例如其位置）、整个动画（例如当前时间）甚至应用的整个生命周期（例如用户偏好设置）的常用值。以下代码创建 uniform 缓冲区，与创建顶点缓冲区非常相似（因为 uniform 会通过与顶点相同的 GPUBuffer 对象传递给 WebGPU API），唯一不同是`GPUBufferUsage.UNIFORM`。

```ts
const uniformArray = new Float32Array([GRID_SIZE, GRID_SIZE]);
const uniformBuffer = device.createBuffer({
  label: "Grid Uniforms",
  size: uniformArray.byteLength,
  usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
});
device.queue.writeBuffer(uniformBuffer, 0, uniformArray);
```

uniform 缓冲区就像是着色器的全局变量，因此可用于所有三种着色器（顶点、片段和计算）。你可以在执行着色器之前设置它们的值，然后在着色器的每次迭代中都使用这些值。在下次 GPU 执行着色器时，你可以将其设置为其他值。

### 1.4 storage buffer(存储缓冲区)

storage 缓冲区也类似于一种全局变量，因为 uniform 缓冲区的大小有限，无法支持动态大小的数组（您必须在着色器中指定数组大小），所以在另一些场景下需要用到存储缓冲区。

存储缓冲区是通用缓冲区，可以在计算着色器中读取和写入，并在顶点着色器中读取。它们可能非常大，并且不需要在着色器中声明特定大小，因此它们更类似于常规内存。这就是您用来存储单元格状态的内容。

> GPU 很可能会对 uniform 缓冲区进行特殊处理，以使它们的更新和读取速度比存储缓冲区更快，因此对于可能会频繁更新且数量较少的数据（例如模型、视图和投影矩阵），uniform 通常是一种更安全的选择，可以实现更好的性能。

以下代码创建了一个存储缓冲区：

```ts
const cellStateArray = new Uint32Array(GRID_SIZE * GRID_SIZE);
// Create a storage buffer to hold the cell state.
const cellStateStorage = device.createBuffer({
  label: "Cell State",
  size: cellStateArray.byteLength,
  usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
});
```

### 1.5 uniform 缓冲区与存储缓冲区的区别

1. uniform 缓冲区在典型的使用情况下速度更快

   这确实取决于用例，一个典型的应用程序是需要绘制大量不同的内容。比方说，这是一款 3D 游戏。应用程序可能会绘制汽车、建筑物、岩石、灌木丛、人物等…每一种都需要传递方向和材质属性，与我们上面的示例所传递的类似。在这种情况下，建议使用统一缓冲区。

2. 存储缓冲区的大小可以比 uniform 缓冲区大得多

   - uniform 缓冲区的最小最大值为 64KiB(65536 bytes)
   - 存储缓冲区的最小最大值为 128MiB(134217728 bytes)

> 所谓的最小最大值，是指某类缓冲区的最大容量。对于 uniform 缓冲区，最大大小至少为 64KiB。对于存储缓冲区，则至少为 128 MiB。

3. 存储缓冲区可读写，uniform 缓冲区只能读

> 从 uniform 到 storage buffer 的转换非常简单，如文章顶部的 storage buffer 所示。vertex buffer 仅用于顶点着色器。它们更为复杂，因为需要向 WebGPU 描述数据布局。texture 最为复杂，因为它们有大量类型和选项。

## 2. 绑定组 (bind group)

不同于顶点缓冲区需要 vertexBufferLayout 与着色器建立关联，uniform 需要使用使用 bind group 建立这种关联。

bind group 是可供着色器访问的资源集合，描述了一组资源以及如何通过着色器访问它们。它可以包含多种类型的缓冲区（例如 uniform 缓冲区）和其他资源（例如纹理和采样器）。

> 填充纹理数据的经典方式是将像素数据先复制到一个缓冲区，然后再从缓冲区复制到纹理中。

```ts
const bindGroup = device.createBindGroup({
  label: "Cell renderer bind group",
  layout: cellPipeline.getBindGroupLayout(0), // 当pipeline的layout字段为auto时
  entries: [
    {
      binding: 0,
      resource: { buffer: uniformBuffer },
    },
  ],
});
```

layout 用来描述此 bind group 包含的资源类型。

entries 数组中，每个条目都是一个至少包含以下值的字典：

- **`binding`**，对应于您在着色器中输入的  `@binding()`  值。在本示例中，该值为  `0`

- **`resource`**：您要向指定绑定索引处的变量公开的实际资源。在本示例中，该资源为 uniform 缓冲区

该函数会返回  `GPUBindGroup`，这是一个不透明的不可变句柄。创建 bind 组后，您便无法更改其指向的资源，但您可以更改这些资源的内容。例如，如果您将 uniform 缓冲区更改为包含新的网格大小，则使用此 bind 组的未来绘制调用会反映这一点。

此外，还需要在 wsgl 中添加如下声明语句：

```wgsl
@group(0) @binding(0) var<uniform> grid: vec2f;
@vertex
fn vertex_main(@location(0) pos: vec2f) ->
  @builtin(position) vec4f {
  return vec4f(pos / grid, 0, 1);
}
```

上述代码在着色器中定义一个名为 grid 的 uniform，它是一个 2D 浮点矢量，与您刚刚复制到 uniform 缓冲区的数组 uniformArray 相匹配，并且指定了 uniform 绑定在`@group(0)`  和  `@binding(0)`  处。之后可以直接使用 grid 的值，该值在每次 vertex_main 被调用时保持一致。注意在 wsgl 中，可以用矢量直接相除，用法同 numpy。`@binding(0)` 与前面 bindGroup 中的 binding 字段相对应。

group(0)与 bind group layout 对应。

### 2.1 绑定组布局

### 2.2 使用绑定组关联缓冲区

在 WGSL 中声明 uniform 和 storage 缓冲区如下，可见这两种缓冲区就像是全局变量一样。不过只能读取，不能写入。group 为 0 说明和之前的 uniform 缓冲区共用一个 bind group。

```wgsl
@group(0) @binding(0) var<uniform> grid: vec2f; // uniform 缓冲区
@group(0) @binding(1) var<storage> cell_state: array<u32>; // storage缓冲区
```

之后需要在 bindGroup 中添加这两个缓冲区的 entry，以便着色器能获取到数据。

```ts
const bindGroup = device.createBindGroup({
  label: "Cell renderer bind group",
  layout: cellPipeline.getBindGroupLayout(0),
  entries: [
    {
      binding: 0,
      resource: { buffer: uniformBuffer },
    },
    {
      binding: 1,
      resource: { buffer: storageBuffer },
    },
  ],
});
```

可以借助于存储缓冲区实现  **乒乓球缓冲区模式**。即创建两个存储缓冲区，一个用于显示的同时在另一个上修改数据。之后切换缓冲区，将上一步修改好的数据用于显示，释放出显示完的数据进行下一步的修改。注意切换操作应放入无限循环中以达到动画效果，建议放在`requestAnimationFrame()`  函数中以与屏幕刷新相同的频率（每秒 60 次）安排回调。

### 2.3 将 bind group 绑定到绘制命令

```ts
pass.setPipeline(cellPipeline);
pass.setVertexBuffer(0, vertexBuffer);
pass.setBindGroup(0, bindGroup); // New line!
pass.draw(vertices.length / 2);
```

作为第一个参数传递的 0 对应于着色器代码中的 `@group(0)`。可视为 `@group(0)` 中的每个 `@binding` 都会使用此 bind group 中的资源。此外，pipelineLayout 中的 bindGroupLayouts 字段的索引也与传入 `@group` 的值对应。
