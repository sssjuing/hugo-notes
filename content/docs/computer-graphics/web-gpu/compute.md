---
title: 在 GPU 上计算
weight: 4
---

- MAP_READ 缓冲区从 GPU 读取数据到 js

![](https://webgpufundamentals.org/webgpu/lessons/resources/webgpu-simple-compute-diagram.svg)

## 1. 数据布局和绑定组

### 1.1 准备缓冲区

### 1.2 将 JS 中的数据写入缓冲区

### 1.3 工作组和工作单元

对于计算着色器的更高级用途，工作组大小变得更加重要。单个工作组内的着色器调用可以共享更快的内存，并使用特定类型的同步基元。但您并不需要这样做，因为着色器执行是完全独立的。

您可以调整工作组大小 (1 x 1 x 1)，这样仍然可以正常工作，但这也会限制 GPU 并行运行着色器的效果。选择更大的大小有助于 GPU 更好地划分工作。

理论上，每个 GPU 都有一个理想的工作组大小，但这取决于 WebGPU 不会公开的架构细节，因此通常情况下，您需要选择一个符合着色器要求的数值。然而，鉴于 WebGPU 内容可运行的各种硬件，64 是一个很好的数字，虽然不太可能超出任何硬件限制，但仍可处理足够大的批次，以达到合理的效率。（8 x 8 == 64，因此您的工作组规模遵循此建议。）

![这是一个工作负载。白边立方体是一个工作项, 红边立方体是一个工作组。](https://surma.dev/assets/workgroups.7887d84b.jpeg)

图中每个红色方块是 workgroup，红色方块中的每个白色基元是一个个的工作项。workgroup_size 则是工作组中以工作项为单位的 3 维尺寸。所有工作项的集合（我称之为『工作负载（workload）』）被分解成若干个工作组（workgroups）。一个工作组中的所有工作项被安排在一起运行。在 WebGPU 中，工作负载被建模为一个 3 维网格，每个『立方体』是一个工作项，工作项组成更大的立方体以形成一个工作组。三者关系为 workload > workgroup > 工作项。

与其他着色器阶段一样，您可以接受各种  `@builtin`  值作为计算着色器函数的输入，以告知您正在进行哪个调用并决定需要执行哪些工作。`global_invocation_id`  内置函数是一个由无符号整数组成的三维矢量，告诉您在着色器调用网格中的位置。您需要对网格中的每个单元格运行一次该着色器。您会得到  `(0, 0, 0)`、`(1, 0, 0)`、`(1, 1, 0)`、…、`(31, 31, 0)`  等数字，这意味着您可以将其视为您要操作的单元格索引！

此外还有 `@builtin(local_invocation_id)`，global_invocation_id 是一个内置值，对应该着色器调用在工作负载中的全局 x/y/z 坐标。 local_invocation_id 是这个着色器在工作组中的 x/y/z 坐标。

![工作负载中标记的三个工作项 a、b 和 c 的示例](https://konata.tech/images/1653570634052.png)

这张图片展示了 @workgroup_size(4, 4, 4) 工作负载坐标系统的一种可能情况。这取决于你为实际使用情况定义什么坐标系。如果我们接受上图中的坐标轴，则 main() 中的 a、b、c 参数如下：

- a: local_id=(x=0, y=0, z=0) global_id=(x=0, y=0, z=0)

- b: local_id=(x=0, y=0, z=0) global_id=(x=4, y=0, z=0)

- c: local_id=(x=1, y=1, z=0) global_id=(x=5, y=5, z=0)

计算着色器也可以像顶点着色器一样使用 uniform 缓冲区和存储缓冲区。上面创建的 cell_state_out 缓冲区用于获取计算着色器的计算结果。

### 1.4 流水线布局及其 bind group

~~`pipelineLayout` 用于描述流水线除了顶点缓冲区外还需要哪些类型的输入。~~ 之前使用的流水线布局是使用  `layout: "auto"` 和 `getBindGroupLayout()`  来获取该布局，此流水线布局是自动创建的。如果您只使用单个流水线，那么这种方法非常有效，但如果您有多个需要共享资源的流水线，则需要显式创建布局，然后将其提供给 bind group 和流水线。因为不同的流水线使用的数据是不同的，或者会以不同的方式使用相同的数据，因此需要多个流水线对相同的几个缓冲区做不同的 bind group 标记。（比如乒乓球缓冲区模式中，渲染流水线在使用一块存储缓冲区进行渲染时，计算流水线会将另一块存储缓冲区用于储存计算计算结果，用当前被渲染的缓冲区作为数据源，当计算和渲染完成后，用 js 代码操作交换两者的用途。）

#### 1.4.1 bind group 布局

bind group 布局用于描述 bind group 中存在的所有资源，可以扩展流水线的定义。绑定组是 GPU 实体（内存缓冲区、纹理、采样器等）的集合，在流水线的执行过程中可以访问。绑定组布局预先定义了这些 GPU 实体的类型、目的和用途，这使得 GPU 能够提前弄清楚如何最有效地运行管线。（因此可以理解为何着色器中@binding 以绑定组为基准，而不是这里的 binding 字段。）

```ts
// Create the bind group layout and pipeline layout.
const bindGroupLayout = device.createBindGroupLayout({
  label: "Cell Bind Group Layout",
  entries: [
    {
      binding: 0,
      visibility: GPUShaderStage.VERTEX | GPUShaderStage.COMPUTE,
      buffer: {}, // Grid uniform buffer
    },
    {
      binding: 1,
      visibility: GPUShaderStage.VERTEX | GPUShaderStage.COMPUTE,
      buffer: { type: "read-only-storage" }, // Cell state input buffer
    },
    {
      binding: 2,
      visibility: GPUShaderStage.COMPUTE,
      buffer: { type: "storage" }, // Cell state output buffer
    },
  ],
});
```

创建 bind group 布局与创建 bind group 本身的结构类似，因为您需要描述  `entries`  的列表。不同之处在于，您需要描述条目必须属于何种资源类型及其使用方式，而不是提供资源本身。

在每个条目中，您都需要提供资源的  `binding`  编号，该编号与着色器中的  `@binding`  值匹配。您还可以提供  `visibility`，这些标志是  `GPUShaderStage`  标记，用于指示哪些着色器阶段可以使用相应资源。

以上代码创建了一个 bind group 布局，标记了三种缓冲区，用于传递数据给顶点着色器和计算着色器，以及从计算着色器获取结果。其中 `@binding(0)` 和 `@binding(2)` 同时作用于顶点着色器和计算着色器，`@binding(2)` 只作用于计算着色器。

在缓冲区字典 `buffer` 字段中，您可以设置相关选项，例如使用缓冲区的 type。默认值为 "uniform"，因此对于绑定 0，您可以将字典留空。（不过，您必须至少设置 buffer: {}，以便条目被标识为缓冲区。）绑定 1 被赋予 "read-only-storage" 类型，因为您不将其与着色器中的 read_write 访问权限一起使用，而绑定 2 具有 "storage" 类型，因为您确实将其与 read_write 访问权限一起使用！**只有这三种 type**。

> 注意：根据下面的代码，他们都属于 `@group(0)`。并且以上代码使用 buffer.type 字段标记出具体是什么缓冲区（GPUBufferUsage.VERTEX / GPUBufferUsage.UNIFORM / GPUBufferUsage.STORAGE），默认值为 `uniform`。具体对应到的缓冲区对象还是和之前一样需要在 bind group 中指定。

在创建好 bind group 布局后，需要在两个地方填入，一个是创建 bind group 时，另一个位置是创建流水线布局时，如下面代码。

```ts
const pipelineLayout = device.createPipelineLayout({
  label: "Cell Pipeline Layout",
  bindGroupLayouts: [bindGroupLayout],
});
```

流水线布局是一个或多个流水线使用的 bind group 布局的列表。数组中 bind group 布局的顺序需要与着色器中的 `@group` 属性相对应。（这意味着 bindGroupLayout 与 `@group(0)` 相关联。）

#### 1.4.2 将 bind group 布局传入 bind group

以下代码将上面创建的 bindGroupLayout 传入了不同的 bindGroup 中，通过交换 binding 实现了在不同的渲染通道上按不同方式使用缓冲区。

```ts
// Create a bind group to pass the grid uniforms into the pipeline
const bindGroups = [
  device.createBindGroup({
    label: "Cell renderer bind group A",
    layout: bindGroupLayout, // Updated Line
    entries: [
      {
        binding: 0,
        resource: { buffer: uniformBuffer },
      },
      {
        binding: 1,
        resource: { buffer: cellStateStorage[0] },
      },
      {
        binding: 2, // New Entry
        resource: { buffer: cellStateStorage[1] },
      },
    ],
  }),
  device.createBindGroup({
    label: "Cell renderer bind group B",
    layout: bindGroupLayout, // Updated Line
    entries: [
      {
        binding: 0,
        resource: { buffer: uniformBuffer },
      },
      {
        binding: 1,
        resource: { buffer: cellStateStorage[1] },
      },
      {
        binding: 2, // New Entry
        resource: { buffer: cellStateStorage[0] },
      },
    ],
  }),
];
```

以上代码中的 binding 值将对应于着色器中传入@binding 的值，而不是 bindGroupLayout 中定义的 binding 值。（但是 bindGroup 和 bindGroupLayout 是通过 binding 字段值映射的。）

## 2. 计算着色器准备

计算着色器与顶点和 fragment 着色器类似，因为它们都设计为在 GPU 上以极端并行性运行，但与其他两个着色器阶段不同，它们没有一组特定的输入和输出。您只从所选来源（例如存储缓冲区）读取数据以及将数据写入所选来源。这意味着，您不必为每个顶点、实例或像素执行一次，而是必须告知您想要调用着色器函数的次数。然后，当您运行着色器时，系统会告知您正在处理哪个调用，并且可以决定要访问哪些数据以及从中执行哪些操作。

```ts
// Create the compute shader that will process the simulation.
const simulationShaderModule = device.createShaderModule({
  label: "Game of Life simulation shader",
  code: `
    @compute
    fn compute_main() {

    }`,
});
```

由于 GPU 经常用于 3D 图形，因此计算着色器的结构可让您请求沿 X 轴、Y 轴和 Z 轴调用特定次数的着色器。这样一来，您就可以轻松调度符合 2D 或 3D 网格的工作，非常适合您的用例！您需要调用该着色器  `GRID_SIZE` x `GRID_SIZE`  次，每个模拟单元格分别调用一次。

鉴于 GPU 硬件架构的性质，此网格分为多个工作组。工作组的大小有 X、Y 和 Z，虽然每个组的大小可以为 1，但扩大工作组的大小通常会有性能方面的优势。对于着色器，可以选择 8 x 8 的任意工作组大小。这对于在 JavaScript 代码中执行跟踪非常有用。

```wgsl
const WORKGROUP_SIZE = 8;

@group(0) @binding(2) var<storage, read_write> cell_state_out: array<u32>;

@compute
@workgroup_size(WORKGROUP_SIZE, WORKGROUP_SIZE) // 输入参数有3个，分别代表X,Y,Z。默认为1
fn compute_main(@builtin(global_invocation_id) cell: vec3u) {

}
```

## 3. 计算流水线

渲染流水线和计算流水线不能混用，因为前者最后会传入 render pass，后者会传入 compute pass。但是 wgsl 可以共用，因为 entry point 可以区分出计算 shader。计算流水线比渲染流水线简单得多，因为计算流水线无需设置任何状态，而只需着色器和布局。

```ts
const simulationPipeline = device.createComputePipeline({
  label: "Simulation pipeline",
  layout: pipelineLayout,
  compute: {
    module: simulationShaderModule,
    entryPoint: "compute_main",
  },
});
```

## 4. 命令编码与计算通道

随后只需要把计算流水线和 bind group 注入计算通道（compute pass）即可。最后，您不像在渲染通道中那样绘制，而是将工作分派给计算着色器，告知它您想要在每个轴上执行多少个工作组。

```ts
const computePass = encoder.beginComputePass();
computePass.setPipeline(simulationPipeline);
computePass.setBindGroup(0, bindGroups[step % 2]);
const workgroupCount = Math.ceil(GRID_SIZE / WORKGROUP_SIZE);
computePass.dispatchWorkgroups(workgroupCount, workgroupCount);
computePass.end();
```

此处需要注意的一点是，您传递到 dispatchWorkgroups() 的数字并不是调用次数，而是要执行的工作组的数量，而工作组的大小由着色器中的 @workgroup_size 定义。如果您希望着色器执行 32x32 次以覆盖整个网格，并且您的工作组大小为 8x8，则需要分派 4x4 个工作组 (4 \* 8 = 32)。因此，需要将网格大小除以工作组大小，然后将该值传递到 dispatchWorkgroups()。

计算工作只能按工作组分派，因此，如果您的工作负载不是工作组大小的偶数因子(not an even divisor of the workgroup size)，则有两种选择。您可以更改工作组大小（此操作必须在着色器模块创建时完成，开销相对较高），也可以将分派的工作组数量向上舍入，然后在着色器中检查是否已超过预期的 global_invocation_id 以及是否需要提前返回。

{{< callout>}}
启动计算流水线要用 `dispatchWorkgroups` ，而不是 `draw`。
{{< /callout >}}

## 5. 读取 GPU 计算结果到 JS

不能直接从 JavaScript 中读取 WebGPU 缓冲区的内容。相反，您必须 “映射 “它，这是从 WebGPU 请求访问缓冲区的另一种方式，因为缓冲区可能正在使用中，而且可能只存在于 GPU 上。

可以在 JavaScript 中映射的 WebGPU 缓冲区不能用于其他用途。换句话说，我们无法映射刚刚创建的缓冲区，如果我们尝试添加标记使其可以映射，就会得到一个与使用 STORAGE 不兼容的错误信息。

因此，为了查看计算结果，我们需要另一个缓冲区。运行计算后，我们将把上面的缓冲区复制到这个结果缓冲区，并设置其标志，以便进行映射。

首先创建一个可写的缓冲区，并将其添加到绑定组，这样它就可以被计算着色器写入。然后创建第二个同样大小的缓冲区作为暂存缓冲区。每个缓冲区的创建都有一个 usage 位掩码，你可以用它声明你打算如何使用这个缓冲区。然后，GPU 会得出满足使用情况的缓冲区的位置，如果无法实现满足标志组合的位置，则会出现错误。

```ts
const BUFFER_SIZE = 1000;
const output = device.createBuffer({
  size: BUFFER_SIZE,
  usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_SRC,
});
const stagingBuffer = device.createBuffer({
  size: BUFFER_SIZE,
  usage: GPUBufferUsage.MAP_READ | GPUBufferUsage.COPY_DST,
});
const bindGroup = device.createBindGroup({
  layout: bindGroupLayout,
  entries: [{ binding: 1,resource: { buffer: output } }],
});
```

使用可写缓存 STORAGE | COPY_SRC 存储数据，用 `CommandEncoder` 的 `copyBufferToBuffer` 方法将内容复制到 MAP_READ | COPY_DST，然后使用宿主代码读取内容）。

```ts
await stagingBuffer.mapAsync(
  GPUMapMode.READ,
  0, // Offset
  BUFFER_SIZE // Length
);
const copyArrayBuffer = stagingBuffer.getMappedRange(0, BUFFER_SIZE);
const data = copyArrayBuffer.slice();
stagingBuffer.unmap();
console.log(new Float32Array(data));
```

stagingBuffer.getMappedRange() 让我们请求将一个区块（或整个缓冲区）作为一个 ArrayBuffer 暴露给 JavaScript。这是实际的、映射的 GPU 内存，这意味着当 stagingBuffer 被解除映射时，数据将消失（ ArrayBuffer 将被『分离』，因为 ArrayBuffer 本质上是个 View），所以我使用 slice() 来创建它在 JavaScript 的副本。
