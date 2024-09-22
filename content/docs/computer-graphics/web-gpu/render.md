---
title: 绘制过程
weight: 2
---

---

![](https://webgpufundamentals.org/webgpu/lessons/resources/webgpu-simple-triangle-diagram.svg)

你向你的 GPU 上传一个数据缓冲区，并告诉它如何将这些数据解释为一系列的三角形。每个顶点都占据了数据缓冲区的一个块，描述该顶点在三维空间中的位置，但也可能包括颜色、纹理 ID、法线和其他数据。列表中的每个顶点都由 GPU 在顶点阶段处理，在每个顶点运行顶点着色器，它将实现平移、旋转或透视变形等操作。（着色器泛指任何在 GPU 上运行的程序。）

之后 GPU 对三角形进行光栅化处理，这意味着 GPU 会计算出每个三角形在屏幕上覆盖的像素。然后，每个像素由像素着色器处理，它可以访问像素坐标，也可以访问辅助数据来决定该像素应该是哪种颜色。

这种将数据传给顶点着色器，然后传给像素着色器，再直接输出到屏幕上的系统被称为管线，在 WebGPU 中，你必须明确地定义你的管线。其中：

- 管道(Pipeline). 它包含 GPU 将运行的顶点着色器和片段着色器。您也可以在管道(Pipeline)中加入计算着色器。
- 着色器通过绑定组(Bind Groups)间接引用资源（缓冲区(buffer)、纹理(texture)、采样器(sampler)）
- 管道定义了通过内部状态间接引用缓冲区的属性
- 属性从缓冲区中提取数据，并将数据输入顶点着色器
- 顶点着色器可将数据输入片段着色器
- 片段着色器通过 render pass description 间接写入纹理

要在 GPU 上执行着色器，需要创建所有这些资源并设置状态。创建资源相对简单。有趣的是，大多数 WebGPU 资源在创建后都无法更改。您可以更改它们的内容，但是无法更改它们的大小、用途、格式等等。如果要更改这些内容，需要创建一个新资源并销毁旧资源。

有些状态是通过创建和执行命令缓冲区来设置的。命令缓冲区顾名思义。它们是一个命令缓冲区。你可以创建编码器。编码器将命令编码到命令缓冲区。编码器*完成*(_finish_)编码后，就会向你提供它创建的命令缓冲区。然后，您就可以*提交*(_submit_)该命令缓冲区，让 WebGPU 执行命令。

目前，WebGPU 允许你创建两种类型的管线。渲染管线和计算管线。顾名思义，渲染管线是指创建一个 2D 图像。该图像不需要在屏幕上，但可以直接渲染到内存中（这被称为帧缓冲区）。计算管道更通用，它返回一个缓冲区，其中可以包含任何类型的数据。

## 0. 绘制准备

```ts
const canvas = document.createElement("canvas");
document.body.appendChild(canvas);
const context = canvas.getContext("webgpu")!;

const adapter = await navigator.gpu.requestAdapter();
const device = await adapter.requestDevice();
const context = canvas.getContext("webgpu")!;
const canvasFormat = navigator.gpu.getPreferredCanvasFormat();
context.configure({ device, format: canvasFormat });
```

从 canvas dom 对象上获取  `GPUCanvasContext` （`rgba8unorm`  或  `bgra8unorm`），之后向  `configure`  方法传入从 adapter 上获取到的 device，及其使用的纹理格式。从而可以将 webgpu 渲染的内容绘制到屏幕上。

此处的纹理格式在构建渲染流水线时还要用，因为要与使用此流水线的所有 render pass 的  `colorAttachments`  中提供的纹理一致。

{{<callout type="info">}}
注意：与 WebGL 相比，WebGPU 的工作方式有一个很大的区别，因为画布配置与设备创建是分开的，因此您可以拥有任意数量的画布，全部由一台设备渲染！这样可以大大简化某些用例（如多窗格 3D 编辑器）的开发工作。
{{</callout>}}

适配器则是操作系统的原生图形 API 与 WebGPU 之间的转换层。由于浏览器是一个单一的操作系统级应用，它可以运行多个 Web 应用，因此又需要进行多路复用，使每个 Web 应用感觉到它对 GPU 有唯一的控制权。这在 WebGPU 中用逻辑设备的概念来模拟。为了获得对一个适配器的访问，你可以调用  `navigator.gpu.requestAdapter()` 。如果有必要，你可以通过  `adapter.limit`  检查物理 GPU 的实际限制，并通过向  `requestDevice()`  传递一个 options 对象来请求一个具有更高限制的 device 。

## 1. 数据准备

### 1.1 顶点缓冲区

GPU 无法直接使用 JavaScript 数组中的数据绘制顶点。GPU 一般都会有自己的专用内存，他针对渲染进行了高度优化，因此，您希望 GPU 在绘制时使用的任何数据都需要放在该内存中。对于 GPU 内存中存储的大量数据，GPU 通过  `GPUBuffer`  对象进行管理，它是 GPU 可以轻松访问的内存块，并且是被标记用于特定用途的内存块。您可以将其视为一个 GPU 可见的 TypedArray。 以下代码创建了顶点缓冲区，其中 vertices 为顶点数据列表，TypedArray 类型。

> javascript 通过 TypedArray (如 Float32Array) 将顶点数据传入 GPU。TypedArray 是一组 JavaScript 对象，可让您分配连续的内存块，并将系列中的每个元素解释为特定数据类型。例如，在 Uint8Array 中，数组中的每个元素都是一个无符号字节。TypedArray 非常适合与对内存布局敏感的 API（例如 WebAssembly、WebAudio 和 WebGPU）来回发送数据。

```ts
const vertexBuffer = device.createBuffer({
  label: "Cell vertices", // 缓冲区标签, 可选, 在报错时方便定位问题
  size: vertices.byteLength, // 缓冲区大小(字节)
  // vertices是JS的TypedArray类型,
  // 其byteLength方法可获取其字节长度
  usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST, // 指示缓冲区的用法
  // GPUBufferUsage.VERTEX指定将该缓冲区用于顶点数据
  // GPUBufferUsage.COPY_DST指示可以将数据复制到其中
});
```

返回的缓冲区对象是不透明的，您无法（轻松地）检查其存储的数据。此外，它的大多数属性都是不可变的，因此您无法在创建 GPUBuffer 后调整其大小，也无法更改用法标志，只可以更改其中的数据（通过  `GPUBufferUsage.COPY_DST`  标志）。因此随后可以通过  `device.queue.writeBuffer`  方法将 JS 的数据复制到缓冲区中，即复制到显存中。

{{<callout type="info">}}
注意：复制后可以修改或删除 vertices 的数据，因为内容已经被复制，不会和原数据关联。所以可以通过重复使用 vertices 达到节省内存的目的。
{{</callout>}}

顶点缓冲区与其他 WebGPU 缓冲区不同的是，我们不直接从顶点着色器访问它们。相反，我们要告诉 WebGPU 缓冲区中的数据种类、位置和组织方式。然后，WebGPU 会从缓冲区中提取数据并提供给我们。

### 1.2 缓冲区布局

顶点缓冲区布局对象定义了缓冲区在内存中的表示方式，render pipeline 需要它来在着色器中映射缓冲区。

对 GPU 而言，上一步创建顶点缓冲区中的数据仅相当于一个字节 Blob，GPU 对此数据是什么以及如何解析仍然一无所知，所以需要通过  `GPUVertexBufferLayout`  告知 GPU 顶点的更多信息，如数据结构等。

```ts
const vertexBufferLayout: GPUVertexBufferLayout = {
  arrayStride: 8,
  stepMode: "vertex",
  attributes: [
    {
      format: "float32x2", // 表示这个顶点数据的属性占用两个float32数据类型
      offset: 0,
      shaderLocation: 0, // 可以是0-15间的任意整数，且对于您定义的每个属性都必须具有唯一性
    },
  ],
};
```

`arrayStride` 字段指示了 GPU 在查找下一个顶点时需要在缓冲区中向后跳过的字节数，其值大小与下面 attributes 有关，相当于所有 attributes 的 format 占用字节之和。

`stepMode` 默认为 `vertex`，表示在顶点着色器每次被调用时以顶点为，即每次调用顶点着色器时步进一个 arrayStride，在绘制新的实例时重新开始。另一个值是 `instance`，表示每个顶点处值相同，每个实例步进一个 arrayStride。

`attributes` 指示了一个 `arrayStride` 的字节上存储的信息，也就是编码到每个顶点中的信息（更高级的用例往往会使用包含多个属性的顶点，例如顶点的颜色或几何图形表面所指向的方向）。注意 attributes 是一个列表，其内部所有成员的总和按顺序标记在一个 arrayStride 的字节上。

`format` 字段的值来自于  `GPUVertexFormat`  类型列表，这些类型描述了 GPU 可以理解的每种顶点数据。

`offset` 字段指示了本条属性相对于当前节点起始处的偏移量，多个属性时需要设置为前面的所有属性占用的字节数之和。

`shaderLocation`  的数据会通过  `@location()`  装饰器，和顶点着色器中的特定参数相关联。这样 WebGPU 会将顶点缓冲区中特定位置的数据注入到顶点着色器的特定输入参数中。

这里可以定义多个顶点缓冲区，对应于 attributes 字段的多个值组成的数组，其中 atrributes 字段中对象的索引号对应于 renderPass.setVertexBuffer(0, vertexBuffer)的第一个参数。多次调用 setVertexBuffer 方法可以传递多个顶点缓冲区。

{{<callout type="warning">}}
注意，顶点着色器不能在多个 render pipeline 间共享。
{{</callout >}}

### 1.3 数据加载到缓冲区

内存对齐

TODO: 待完善

```ts
device.queue.writeBuffer(vertexBuffer, /*bufferOffset=*/ 0, vertices);
```

## 2. 着色器

着色器是您编写并在 GPU 上执行的小程序。每个着色器都在数据的不同阶段运行：顶点处理、Fragment 处理或常规计算。由于它们基于 GPU，因此结构比一般 JavaScript 更严格。但这种结构使它们能够非常快速地执行，最重要的是并行执行！

可以通过调用  `createShaderModule`  创建一个着色器模块，其内部需要填入 label 和 code 两个字段，label 和之前创建 buffer 时一样，用于在报错时提供定位信息，code 为 wsgl 代码。函数返回一个  `GPUShaderModule`  对象。

```ts
const cellShaderModule = device.createShaderModule({
  label: "Cell shader",
  code: `
    // Your shader code will go here
  `,
});
```

### 2.1 顶点着色器

关于顶点着色器，请务必注意，系统不一定会按顺序调用这些函数。相反，GPU 擅长并行运行这类着色器，有可能同时处理数百个（甚至数千个）顶点！这是 GPU 实现惊人速度的重要一环，但也存在局限性。为了确保实现极致的并行处理，顶点着色器不能相互通信。着色器调用一次只能查看一个顶点的数据，并且只能输出单个顶点的值。

在 WGSL 中，顶点着色器函数可以随意命名，但它前面必须有 @vertex 属性，以便指明它所代表的着色器阶段。顶点着色器必须至少返回在裁剪空间中处理的顶点的最终位置。始终以四维矢量的形式指定。矢量在着色器中非常常用，因此在语言中被视为一流基元，具有自己的类型，例如四维矢量的 vec4f。2D 矢量 (vec2f) 和 3D 矢量 (vec3f) 也有类似的类型！`@builtin(position)`  属性标记了此字段将作为顶点在裁剪坐标系中的位置来使用。这类似于 GLSL 的  `gl_Position`  变量。

```wgsl
@vertex
fn vertex_main(@location(0) pos: vec2f) ->
  @builtin(position) vec4f {
  return vec4f(pos, 0, 1); // 相当于 return vec4f(pos.x, pos.y, 0, 1);
}
```

在上一步中指定的 vertexBufferLayout 中，定义了 shaderLocation 的值为 0，对应于这里的  `@location(0)`  修饰符，因此 vertexBuffer 的值以两个 float32 为一组传入此处的 pos 中，注意 pos 的类型 vec2f 对应了  `vertexBufferLayout`  中的  `float32x2`。顶点着色器还可以传入`@builtin(vertex_index)`，用于获取顶点的索引。

{{<callout type="info">}}
注意：到目前为止 `vertexBuffer`、`vertexBufferLayout`、`cellShaderModule` 三者还没有在代码中关联起来，这些内容以及后续的 `bindGroup` 和 `pipelineLayout`  将在构建 pipeline 时关联在一起。
{{</callout>}}

### 2.2 fragment 着色器

{{<callout type="info">}}
fragment 着色器对应于计算机图形学中的光栅化(rasterizing)步骤。
{{</callout>}}

fragment 着色器的绘制模式有以下 5 种，在 `createRenderPipeline` 创建流水线时用 `primitive.topology` 指定：

- `'point-list'`: 对于每个顶点，绘制一个点
- `'line-list'`: 每 2 个点绘制一条线
- `'line-strip'`: 绘制最新点与前一点的连接线
- `'triangle-list'`: 每 3 个点绘制一个三角形 (**默认**)
- `'triangle-strip'`: 对于每个新位置，从它和最后 2 个位置中画出一个三角形

不同于顶点着色器，Fragment 着色器针对要绘制的每个像素调用。Fragment 着色器始终在顶点着色器之后调用。GPU 获取顶点着色器的输出并对其进行三角形处理，从而基于三个点的集合创建三角形。然后，它会计算出每个三角形都包含输出颜色附件的哪些像素，从而对每个三角形进行光栅化，然后针对每个像素调用一次 fragment 着色器。fragment 着色器会返回一种颜色，通常根据从顶点着色器发送到它的值以及 GPU 将其写入颜色附件的纹理等资源计算得出。

与顶点着色器一样，fragment 着色器也是以大规模并行方式执行的。在输入和输出方面，它们比顶点着色器更灵活，但您可以认为它们只是为每个三角形的每个像素返回一种颜色。

```wgsl
@fragment
fn fragment_main() -> @location(0) vec4f {
  return vec4f(1, 0, 0, 1); // (Red, Green, Blue, Alpha)
}
```

WGSL fragment 着色器函数由 @fragment 属性表示，并且它也会返回 vec4f。但在这种情况下，矢量表示颜色，而非位置。矢量的四个组成部分是红色、绿色、蓝色和 Alpha 颜色值。

需要为返回值提供 @location 属性，以指示要将返回的颜色写入 beginRenderPass 调用中的哪个 colorAttachment。colorAttachments 是调用 beginRenderPass 时传入的一个数组，字面意思是颜色附件。传入@location 的数字代表 colorAttachments 数组的索引。

{{<callout type="info">}}
注意：如果需要，您还可以为顶点着色器和 fragment 着色器创建单独的着色器模块。例如，如果您想将多个不同的 fragment 着色器与同一个顶点着色器搭配使用，便可利用此功能。
{{</callout>}}

TODO: 怎么使用，添加用例

### 2.3 在顶点着色器在返回位置的同时附带其他数据

顶点着色器可以在每个位置输出额外的值，默认情况下，这些值将在 3 个点之间进行插值。

{{<callout type="info">}}
注意，进行插值的是用@location 标记的值。
{{</callout>}}

返回位置始终需要顶点着色器，因此，如果希望在返回位置的同时返回任何其他数据，则需要将其放入一个结构体中。WGSL 中的结构体是包含一个或多个命名属性的命名对象类型。这些属性也可以使用 @builtin 和 @location 等属性进行标记。您可以在任何函数外声明这些属性，然后根据需要将这些属性的实例传入函数以及从函数中传出。

这种在 wgsl 中的着色器之间进行数据传递的变量为 Inter-stage 变量，最常用于在三角形内进行纹理坐标插值。与 WebGPU 中的几乎所有功能一样，顶点着色器和片段着色器之间的 Inter-stage 变量是通过索引（@location）连接的。这种连接即使 wgsl 在不同的 shader module 中也是成立的。

注意区分内置变量@builtin。@builtin(position) 在顶点着色器和片段着色器中的意义是不同的。在顶点着色器中，@builtin(position) 是 GPU 绘制三角形/线/点所需的输出。在片段着色器中，@builtin(position) 是一个输入。它是片段着色器当前被要求计算颜色的像素坐标。

```wgsl
struct VertexInput {
  @location(0) pos: vec2f,
};

struct VertexOutput {
  @builtin(position) pos: vec4f,
  @location(0) cell: vec2f, // New line!
};

@group(0) @binding(0) var<uniform> grid: vec2f;

@vertex
fn vertex_main(input: VertexInput) -> VertexOutput  {
  var output: VertexOutput;
  output.pos = vec4f(gridPos, 0, 1);
  output.cell = vec2f(0, 1);
  return output;
}
```

> 在输出结构体中用  `@builtin(position)`  修饰位置向量（vec4f 类型），用  `@location(0)`  标记其他的数据，这部分数据可在后续的片元着色器中用相同的  `@location(0)`  捕获到（参数名称不要求匹配，但建议一致，便于跟踪）。

Inter-stage 变量从顶点着色器的输出在传递给片段着色器时会进行插值。对于如何进行插值，有两组设置可以更改。将它们设置为默认值以外的值并不常见。

插值类型:

- `perspective`: 以正确的透视方式插值 (**默认**)
- `linear`: 以线性、非透视正确的方式内插数值。
- `flat`: 不对数值进行插值。使用 flat 插值时不使用插值采样。

插值采样:

- `center`: 插值在像素中心进行 (**默认**)
- `centroid`: 插值是在当前基元中片段所覆盖的所有样本内的某一点上进行的。该值对基元中的所有样本都是相同的。
- `sample`: 每次采样时执行插值。应用此属性时，每次采样都会调用一次片段着色器.

```wgsl
@location(2) @interpolate(linear, center) myVariableFoo: vec4f;
@location(3) @interpolate(flat) myVariableBar: vec4f;
```

### 2.4 在两种着色器之间传递数据

可以将片元着色器的输入参数声明为 VertexOutput，这样就可以捕获顶点着色器传出的数据了。

{{<callout type="warning">}}
注意：默认情况下，这些值将在 3 个点之间进行插值，进行插值的是用 `@location` 标记的值。
{{</callout>}}

```wgsl
@fragment
fn fragment_main(input: VertexOutput) -> @location(0) vec4f {
  return vec4f(input.cell, 0, 1);
}
```

当然也可以再定义一个 FragInput 结构体，但要注意将对应的字段用@location(0)修饰才能获取数据。

```wgsl
struct FragInput {
  @location(0) cell: vec2f,
};
```

## 3. 流水线

上述创建的缓存(vertexBuffer)、布局(vertexBufferLayout、pipelineLayout)和着色器模块(cellShaderModule)需要传入渲染流水线才能交给 GPU 进行渲染。通过调用 device 的  `createRenderPipeline`  方法可创建一个渲染流水线。

渲染流水线可以控制几何图形的绘制方式，包括使用哪些着色器、如何解读顶点缓冲区中的数据、应渲染哪种类型的几何图形（线、点、三角形…），等等！渲染流水线是整个 API 中最复杂的对象，但不必担心，您可以传递给它的大多数值都是可选的，并且只需要提供几个值即可开始使用。

```ts
const cellPipeline = device.createRenderPipeline({
  label: "Cell pipeline",
  layout: "auto",
  vertex: {
    module: cellShaderModule,
    entryPoint: "vertex_main",
    buffers: [vertexBufferLayout],
  },
  fragment: {
    module: cellShaderModule,
    entryPoint: "fragment_main",
    targets: [{ format: canvasFormat }],
  },
});
```

其中`pipelineLayout`、`cellShaderModule`、`vertexBufferLayout`均为之前创建，canvasFormat 为准备 canvas context 时创建的，此处传入的应和之前传入  `context.configure`  方法中的保持一致，如前所述。

当 layout 设置为`‘auto'`时，将会使流水线根据您在着色器代码本身中声明的绑定自动创建 bind group 布局（要求 WebGPU 从着色器中推断出数据布局）。 ‘auto’很方便，但使用 layout: ‘auto’ 布局无法在不同管道中共享绑定组。

vertex 字段需要提供有关 vertex 的详细信息。包括着色器、着色器代码中为每个顶点调用的函数的名称(entryPoint)、描述数据如何打包到用于此流水线的顶点缓冲区中（ `GPUVertexBufferLayout`  对象的数组）。buffers 字段可放置多个顶点缓冲区布局，实际的顶点缓冲区在渲染通道中通过 `setVertexBuffer` 传入。

fragment 字段提供有关 fragment 的详细信息。module 和 entryPoint 同上，targets 用于定义此流水线搭配使用的 targets，里面的信息要与使用此流水线的所有渲染通道的 colorAttachments 中提供的纹理一致。

## 4. 通道

要发送到 GPU 的命令与渲染相关，需要使用 encoder 启动渲染通道。渲染通道是指 WebGPU 中的所有绘图操作发生时需要执行的命令。每个通道都以  `beginRenderPass()`  调用开始，该调用定义接收所执行的任何绘图命令输出的纹理。更高级的用途可以提供多种纹理（称为“Attachment”）来实现各种用途，例如存储所渲染几何图形的深度或提供抗锯齿效果。

{{<callout type="info">}}

注意：上面定义的缓冲区、绑定组等内容已经被收集到了 pipeline 中，这里还要传入一次。

{{</callout>}}

```ts
// 获取一个用于记录 GPU 命令的接口
const encoder = device.createCommandEncoder();
const pass = encoder.beginRenderPass({
  colorAttachments: [
    {
      view: context.getCurrentTexture().createView(),
      clearValue: { r: 0, g: 0, b: 0.4, a: 1.0 },
      // loadOp和storeOp用于定义渲染通道在开始和结束时对纹理执行的操作
      loadOp: "clear", // 在渲染通道开始时用clearValue清除纹理
      // 另一个选项是load，将纹理的现有内容加载到GPU中，以便在已有内容上绘制
      storeOp: "store", // 在渲染通道完成后，将渲染通道期间完成的所有绘制结果保存到纹理
      // 另一个选项是discard
    },
  ],
});
```

传入  `beginRenderPass()`  的对象的类型为  `GPURenderPassDescriptor`，它描述了我们要绘制的纹理以及如何使用它们。`render pass`  编码器是一种特定的编码器，用于创建与渲染相关的命令。我们将 renderPassDescriptor 传递给它，告诉它我们要渲染到哪个纹理。

clearValue 会指示渲染通道在通道开始时执行 clear 操作时应使用哪种颜色。向其中传递的字典包含四个值：r 表示红色，g 表示绿色，b 表示蓝色，a 表示 alpha（透明度）。每个值的范围介于 0 到 1 之间。

纹理作为 colorAttachment 的 view 属性指定。通过调用 context.getCurrentTexture() 从您之前创建的画布上下文中获取纹理，该方法会返回一个像素宽度和高度与画布的 width 和 height 属性以及调用 context.configure() 时指定的 format 相匹配的纹理。渲染通道要求您提供 GPUTextureView（而非 GPUTexture）来告知它要渲染到纹理的哪些部分。这对于更高级的用例才真正重要，因此在这里您要对纹理调用 createView()（不添加任何参数），这表示您希望渲染通道使用整个纹理。

```ts
// 使用哪个流水线进行绘制
// 包括所使用的着色器、顶点数据的布局以及其他相关状态数据
pass.setPipeline(cellPipeline);
pass.setVertexBuffer(0, vertexBuffer); // 设置顶点缓冲区
pass.draw(vertices.length / 2);
```

`setPipeline` 用于设置使用的流水线，WebGPU 支持多个流水线同时绘制，每个流水线分别绘制不同的类型的图元(primitive，点、线、三角形等)、深度缓冲（depthStencil）等。如果要绘制多个流水线可以多次调用。

`setVertexBuffer` 用于设置多个顶点缓冲区，第一个参数对应于传入 `createRenderPipeline` 的 vertex.buffers 属性的 `GPUVertexBufferLayout` 对象数组的索引，第二个参数是该 `GPUVertexBufferLayout` 对象对应的顶点缓冲区。

`draw` 告诉 WebGPU 绘制此流水线，第一个参数是顶点数量，或称顶点着色器的调用次数。此处因为每个点两个坐标，因此除以 2。还有第二个参数，用于指示这个顶点缓冲区要整体绘制多少遍。

```ts
// End the render pass and submit the command buffer
pass.end();
device.queue.submit([encoder.finish()]);
```

encoder.finish()返回  `GPUCommandBuffer`，它是所记录命令的不透明句柄。之后使用 GPUDevice 的 queue 将命令缓冲区提交到 GPU。队列会执行所有 GPU 命令，确保这些命令的执行顺序合理且适当同步。队列的 submit() 方法接受命令缓冲区数组。命令缓冲区一旦提交，便无法再次使用，因此无需保留。

将命令提交到 GPU 之后，让 JavaScript 将控制权交还给浏览器。这时，浏览器就会发现您已经更改了上下文的当前纹理，并更新画布以将该纹理显示为图片。在此之后，如果您想再次更新画布内容，则需要记录并提交新的命令缓冲区，再次调用  `context.getCurrentTexture()`  以获取用于渲染通道的新纹理。

## 5. 命令缓冲区

由于 GPU 可以是一个完全独立的显卡，有自己的内存芯片，你可以通过所谓的命令缓冲区或命令队列来控制它。命令队列是一大块内存，其中包含供 GPU 执行的已编码的命令。编码方式与 GPU 高度相关，所以这将由驱动程序来处理。WebGPU 暴露了一个  `CommandEncoder` ，来使用该功能。

`CommandEncoder`  有多个方法，允许你将数据从一个 GPU 缓冲区复制到另一个缓冲区和操作纹理。它还允许你创建  `PassEncoder`  对管道的设置和调用进行编码。`PassEncoder`  是 WebGPU 的抽象，用来避免开头提到的 WebGL 的内部全局状态对象。所有运行 GPU 流水线所需的数据和状态都明确地通过 PassEncoder 传递。

## 6. 绘制多个实例

对于要在不同位置绘制多个相同几何图形的问题，一种方法是将实体坐标写入 uniform 缓冲区，然后对每个实体调用 draw 一次，每次都更新 uniform。不过，这会非常慢，因为 GPU 每次都必须等待 JavaScript 写入新坐标。使 GPU 获得良好性能的一个关键点是最大限度减少 GPU 在等待系统其他部分上所花的时间！不过，您可以使用一种称为“实例化”的技术。实例化是一种告知 GPU 通过单次调用 draw 绘制同一几何图形的多个副本的方式，这种方式比针对每个副本调用一次 draw 快得多。几何图形的每个副本都称为一个实例。

实现方式是在调用 pass.draw 方法时传入第二个参数，其内部机制是将顶点缓冲区中的内容重复绘制的次数。但此时会将这些副本绘制在同一个位置上，因此无法显示出全貌。所幸顶点着色器提供了内置值，可以获取到副本的索引。

```wgsl
@group(0) @binding(0) var<uniform> grid: vec2f;

@vertex
fn vertex_main(@location(0) pos: vec2f,
              @builtin(instance_index) instance: u32) ->
  @builtin(position) vec4f {

  let i = f32(instance); // Save the instance_index as a float
  let cell = vec2f(i, i);
  let cellOffset = cell / grid * 2; // Updated
  let gridPos = (pos + 1) / grid - 1 + cellOffset;

  return vec4f(gridPos, 0, 1);
}
```

上面代码中 instance_index 就是上面所说的索引，需要使用  `@builtin(instance_index)`  标记后才能获取。instance_index 从 0 开始，u32 类型，因此需要转换为 f32 才能与浮点数进行计算。

> 注意：循环的策略是  `@location(0)`  在内层循环上递增，`@builtin(instance_index)`  在外层循环上递增。即先按顶点缓冲区的数据依次调用 vertex_main，此时  `@location(0)`  不停变化而  `@builtin(instance_index)`  始终是 0，之后  `@builtin(instance_index)`  递增到 1 并执行上一次的调用。如此往下。

## 7. 使用索引缓冲区

上面的例子使用了重复的顶点数据绘制三角形，当顶点过多时这种方式会在顶点缓冲区存储大量重复的顶点数据（坐标、颜色等）。实际使用时通过使用索引缓冲区，您无需重复顶点数据即可生成三角形。您可以向 GPU 提供单独的值列表，告知将哪些顶点连接成三角形，这样它们就不需要重复了。这就像连接点一样！以下代码创建了索引缓冲区。

```ts
let indexBuffer = device.createBuffer({
  label: "Index buffer",
  size: indices.byteLength,
  usage: GPUBufferUsage.INDEX | GPUBufferUsage.COPY_DST,
});
```

与直接使用顶点缓冲区绘制不同，使用索引缓冲区需要调用 renderPass 的`drawIndexed`方法进行绘制。

```ts
renderPass.setVertexBuffer(0, vertexBuffer);
renderPass.setIndexBuffer(indexBuffer, "uint16");
renderPass.drawIndexed(indices.length);
```

{{<callout type="info">}}
如果想使用存储缓冲区配合索引缓冲区实现类似的效果，需要在顶点着色器中使用 `@builtin(vertex_index)` 内置函数获取当前顶点的索引。
{{</callout>}}

## 8. 使用纹理

纹理通常代表 2d 图像，二维图像只是一个由颜色值组成的二维数组。与直接使用存储缓冲区作为二维数组相比，纹理的特殊之处在于，它们可以被称为*采样器*的特殊硬件访问。采样器可以读取纹理中最多 16 个不同的值，并将它们混合在一起，这对许多常见的使用情况都很有用。

### 8.1 着色器

```wgsl
struct OurVertexShaderOutput {
  @builtin(position) position: vec4f,
  @location(0) texcoord: vec2f,
};

@vertex fn vs(
  @builtin(vertex_index) vertexIndex : u32
) -> OurVertexShaderOutput {
  let pos = array(
    // 1st triangle
    vec2f( 0.0,  0.0),  // center
    vec2f( 1.0,  0.0),  // right, center
    vec2f( 0.0,  1.0),  // center, top

    // 2st triangle
    vec2f( 0.0,  1.0),  // center, top
    vec2f( 1.0,  0.0),  // right, center
    vec2f( 1.0,  1.0),  // right, top
  );

  var vsOutput: OurVertexShaderOutput;
  let xy = pos[vertexIndex];
  vsOutput.position = vec4f(xy, 0.0, 1.0);
  vsOutput.texcoord = xy;
  return vsOutput;
}

@group(0) @binding(0) var ourSampler: sampler;
@group(0) @binding(1) var ourTexture: texture_2d<f32>;

@fragment fn fs(fsInput: OurVertexShaderOutput) -> @location(0) vec4f {
  return textureSample(ourTexture, ourSampler, fsInput.texcoord);
}
```

首先，需要在顶点着色器中生成顶点的纹理坐标，并通过将其标记为 `@location(0)` 传递给 fragment 着色器。`vsOutput.texcoord` 的值在传递给片段着色器时将会在三角形的三个顶点中进行插值。

然后，我们声明了采样器和纹理，并在片段着色器中引用了它们。函数  `textureSample`  对纹理进行采样。第一个参数是要采样的纹理。第二个参数是采样器，用于指定如何对纹理进行采样。第三个参数是纹理坐标，用于指定采样位置。

> 注：将位置值作为纹理坐标传递的做法并不常见，但在这种单位四边形（宽和高各为一个单位的四边形）的特殊情况下，我们需要的纹理坐标恰好与位置值相匹配。这样做可以使示例更小更简单。通过[顶点缓冲区](https://webgpufundamentals.org/webgpu/lessons/webgpu-vertex-buffers.html)提供纹理坐标要常见得多。

TODO: colorAttachments、getCurrentTexture、createView
