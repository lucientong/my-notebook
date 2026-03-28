# GPU 架构与 CUDA 编程

---

## 📑 目录

### GPU 架构
1. [GPU vs CPU 架构对比](#1-gpu-vs-cpu-架构对比)
2. [NVIDIA GPU 架构演进](#2-nvidia-gpu-架构演进)
3. [SM 结构详解](#3-sm-结构详解)
4. [GPU 内存层次](#4-gpu-内存层次)

### CUDA 编程
5. [CUDA 编程模型](#5-cuda-编程模型)
6. [CUDA 内存模型](#6-cuda-内存模型)
7. [同步与原子操作](#7-同步与原子操作)

### 性能优化
8. [CUDA 性能优化](#8-cuda-性能优化)
9. [高级特性](#9-高级特性)
10. [面试题自查](#10-面试题自查)

---

## 1. GPU vs CPU 架构对比

### 1.1 设计哲学

```
CPU：延迟优化（Latency-Oriented）
- 少量强大核心（4-64）
- 大缓存（L1/L2/L3 共几十MB）
- 复杂控制逻辑（分支预测、乱序执行）
- 目标：单线程尽可能快

GPU：吞吐量优化（Throughput-Oriented）
- 大量简单核心（数千）
- 小缓存，大寄存器文件
- 简单控制逻辑
- 目标：大量并行任务的总吞吐
```

### 1.2 架构对比图

```
CPU 架构：                           GPU 架构：
┌─────────────────────────┐         ┌─────────────────────────────────┐
│  ┌─────┐  ┌─────┐       │         │ ┌───┐┌───┐┌───┐┌───┐...┌───┐   │
│  │Core │  │Core │       │         │ │SM ││SM ││SM ││SM │   │SM │   │
│  │ 0   │  │ 1   │       │         │ └───┘└───┘└───┘└───┘   └───┘   │
│  └──┬──┘  └──┬──┘       │         │ ┌───┐┌───┐┌───┐┌───┐...┌───┐   │
│     │        │          │         │ │SM ││SM ││SM ││SM │   │SM │   │
│  ┌──┴────────┴──┐       │         │ └───┘└───┘└───┘└───┘   └───┘   │
│  │   L3 Cache   │       │         │           ...                   │
│  │   (数十MB)   │       │         │                                 │
│  └──────┬───────┘       │         │ ┌─────────────────────────────┐ │
│         │               │         │ │        L2 Cache             │ │
└─────────┼───────────────┘         │ └─────────────────────────────┘ │
          │                         │              │                   │
    ┌─────┴─────┐                   │   ┌─────────┴─────────┐         │
    │   DRAM    │                   │   │   HBM / GDDR      │         │
    │  DDR4/5   │                   │   │  高带宽显存        │         │
    └───────────┘                   └───┴───────────────────┴─────────┘

CPU: 4-64 核，3GHz+，缓存优化
GPU: 数千核，1-2GHz，带宽优化
```

### 1.3 适用场景

| 任务类型 | CPU | GPU |
|---------|-----|-----|
| 串行逻辑 | ✓✓✓ | ✗ |
| 分支密集 | ✓✓✓ | ✗ |
| 低延迟响应 | ✓✓✓ | ✗ |
| 大规模并行计算 | ✗ | ✓✓✓ |
| 矩阵运算 | ✗ | ✓✓✓ |
| 深度学习 | ✗ | ✓✓✓ |

---

## 2. NVIDIA GPU 架构演进

### 2.1 架构世代

| 架构 | 年份 | 代表产品 | 关键特性 |
|------|------|---------|---------|
| Kepler | 2012 | GTX 680, K80 | Dynamic Parallelism |
| Maxwell | 2014 | GTX 980 | 能效优化 |
| Pascal | 2016 | GTX 1080, P100 | NVLink, HBM2 |
| Volta | 2017 | V100 | Tensor Core 首发 |
| Turing | 2018 | RTX 2080 | RT Core, INT8 |
| Ampere | 2020 | A100, RTX 30 | TF32, 结构化稀疏 |
| Hopper | 2022 | H100 | Transformer Engine, FP8 |
| Blackwell | 2024 | B100/B200 | 更大 Tensor Core |

### 2.2 H100 规格（Hopper）

```
H100 SXM5 规格：

SM 数量：132
CUDA Cores：16,896 (128 per SM)
Tensor Cores：528 (4 per SM)

内存：
- HBM3: 80GB
- 带宽: 3.35 TB/s

互联：
- NVLink 4.0: 900 GB/s 双向
- PCIe 5.0: 128 GB/s

性能：
- FP32: 67 TFLOPS
- TF32: 989 TFLOPS (Tensor)
- FP16: 1,979 TFLOPS (Tensor)
- FP8: 3,958 TFLOPS (Tensor)
- INT8: 3,958 TOPS
```

---

## 3. SM 结构详解

### 3.1 Streaming Multiprocessor

```
SM (Streaming Multiprocessor) 结构 - Ampere A100：

┌─────────────────────────────────────────────────────────────────────┐
│                              SM                                     │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────┐  ┌─────────────────────────┐          │
│  │    Warp Scheduler 0     │  │    Warp Scheduler 1     │          │
│  │    Dispatch Unit ×2     │  │    Dispatch Unit ×2     │          │
│  └─────────────────────────┘  └─────────────────────────┘          │
│  ┌─────────────────────────┐  ┌─────────────────────────┐          │
│  │    Warp Scheduler 2     │  │    Warp Scheduler 3     │          │
│  │    Dispatch Unit ×2     │  │    Dispatch Unit ×2     │          │
│  └─────────────────────────┘  └─────────────────────────┘          │
├─────────────────────────────────────────────────────────────────────┤
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                    Register File (256KB)                      │ │
│  │                    65,536 × 32-bit Registers                  │ │
│  └───────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐┌─────────────┐┌─────────────┐┌─────────────┐      │
│  │  FP32 ×16   ││  FP32 ×16   ││  FP32 ×16   ││  FP32 ×16   │      │
│  │  INT32 ×16  ││  INT32 ×16  ││  INT32 ×16  ││  INT32 ×16  │      │
│  └─────────────┘└─────────────┘└─────────────┘└─────────────┘      │
│  ┌─────────────┐┌─────────────┐┌─────────────┐┌─────────────┐      │
│  │ FP64/FP32×8 ││ FP64/FP32×8 ││ FP64/FP32×8 ││ FP64/FP32×8 │      │
│  └─────────────┘└─────────────┘└─────────────┘└─────────────┘      │
│  ┌─────────────┐┌─────────────┐┌─────────────┐┌─────────────┐      │
│  │ Tensor Core ││ Tensor Core ││ Tensor Core ││ Tensor Core │      │
│  └─────────────┘└─────────────┘└─────────────┘└─────────────┘      │
│  ┌─────────────┐                                                    │
│  │  LD/ST ×32  │ Load/Store Units                                  │
│  └─────────────┘                                                    │
│  ┌─────────────┐                                                    │
│  │  SFU ×4     │ Special Function Units (sin, cos, exp...)         │
│  └─────────────┘                                                    │
├─────────────────────────────────────────────────────────────────────┤
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │             Shared Memory / L1 Cache (192KB)                  │ │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Warp 执行模型

```
Warp（线程束）：

- 1 Warp = 32 个线程
- Warp 是 GPU 调度的基本单位
- 同一 Warp 内的线程执行相同指令（SIMT）

执行示意：
┌─────────────────────────────────────────────────────────────────┐
│  Warp 0: Thread 0-31                                            │
│  [T0][T1][T2][T3]...[T31]  →  同时执行 ADD 指令                 │
│                                                                 │
│  Warp 1: Thread 32-63                                           │
│  [T32][T33][T34]...[T63]  →  同时执行 MUL 指令                  │
│                                                                 │
│  Warp 调度器轮流调度不同 Warp 执行                              │
│  当一个 Warp 等待内存时，切换到另一个 Warp 执行                 │
│  隐藏内存延迟！                                                 │
└─────────────────────────────────────────────────────────────────┘

Warp Divergence（分支发散）：
if (threadIdx.x < 16) {
    // 分支 A
} else {
    // 分支 B
}

→ Warp 必须串行执行两个分支（一半线程空闲）
→ 性能下降 50%
```

### 3.3 Tensor Core

```
Tensor Core：矩阵乘加加速单元

单个 Tensor Core 每周期执行：
D = A × B + C

其中 A, B, C, D 是小矩阵

Ampere Tensor Core (TF32 模式)：
  A: 8×4 TF32
  B: 4×8 TF32
  C: 8×8 FP32
  → D: 8×8 FP32

性能对比：
- CUDA Core FP32: 一次乘加（2 FLOPS）
- Tensor Core: 一次 8×4×8 矩阵乘加（1024 FLOPS）

支持的精度：
| 输入  | 累加  | 架构      |
|-------|-------|-----------|
| FP16  | FP32  | Volta+    |
| BF16  | FP32  | Ampere+   |
| TF32  | FP32  | Ampere+   |
| FP8   | FP32  | Hopper+   |
| INT8  | INT32 | Turing+   |
| INT4  | INT32 | Ampere+   |
```

---

## 4. GPU 内存层次

### 4.1 内存层次结构

```
GPU 内存层次（由快到慢）：

┌─────────────────────────────────────────────────────────────────────┐
│  Registers (寄存器)                                                 │
│  - 最快：~1 周期                                                    │
│  - 每 SM 约 256KB，每线程可分配 255 个                             │
│  - 私有，线程内使用                                                 │
├─────────────────────────────────────────────────────────────────────┤
│  Shared Memory (共享内存)                                           │
│  - 快：~30 周期                                                     │
│  - 每 SM 最多 164KB (Ampere)                                       │
│  - Block 内线程共享                                                 │
├─────────────────────────────────────────────────────────────────────┤
│  L1 Cache / Texture Cache                                           │
│  - 中：~30-100 周期                                                 │
│  - 与 Shared Memory 共享物理空间                                   │
│  - 自动缓存 Global Memory 读取                                      │
├─────────────────────────────────────────────────────────────────────┤
│  L2 Cache                                                           │
│  - 较慢：~200 周期                                                  │
│  - 全 GPU 共享（A100: 40MB, H100: 50MB）                           │
├─────────────────────────────────────────────────────────────────────┤
│  Global Memory (HBM/GDDR)                                           │
│  - 慢：~400-800 周期                                                │
│  - 容量大（A100: 80GB）                                             │
│  - 带宽高（A100: 2TB/s）但延迟高                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 内存带宽

| 类型 | 带宽 | 延迟 |
|------|------|------|
| Register | ~20 TB/s | 1 cycle |
| Shared Memory | ~15 TB/s | ~30 cycles |
| L2 Cache | ~5 TB/s | ~200 cycles |
| HBM3 (H100) | 3.35 TB/s | ~400 cycles |

---

## 5. CUDA 编程模型

### 5.1 线程层次

```
CUDA 线程层次：

Grid（网格）
├── Block 0,0    Block 1,0    Block 2,0    ...
│   ├── Thread 0,0
│   ├── Thread 1,0
│   ├── ...
│   └── Thread 31,0
├── Block 0,1    Block 1,1    Block 2,1    ...
└── ...

┌─────────────────────────────────────────────────────────────────────┐
│  Grid                                                               │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                   │
│  │   Block     │ │   Block     │ │   Block     │ ...               │
│  │ ┌───┬───┬───┤ │ ┌───┬───┬───┤ │ ┌───┬───┬───┤                   │
│  │ │T00│T01│T02│ │ │T00│T01│T02│ │ │T00│T01│T02│                   │
│  │ ├───┼───┼───┤ │ ├───┼───┼───┤ │ ├───┼───┼───┤                   │
│  │ │T10│T11│T12│ │ │T10│T11│T12│ │ │T10│T11│T12│                   │
│  │ └───┴───┴───┘ │ └───┴───┴───┘ │ └───┴───┴───┘                   │
│  └─────────────┘ └─────────────┘ └─────────────┘                   │
└─────────────────────────────────────────────────────────────────────┘

内置变量：
- threadIdx.x/y/z：Block 内线程索引
- blockIdx.x/y/z：Grid 内 Block 索引
- blockDim.x/y/z：Block 维度
- gridDim.x/y/z：Grid 维度
```

### 5.2 Kernel 基础

```cpp
// Kernel 定义
__global__ void vectorAdd(float* a, float* b, float* c, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) {
        c[i] = a[i] + b[i];
    }
}

// Host 代码
int main() {
    int n = 1000000;
    float *d_a, *d_b, *d_c;
    
    // 分配设备内存
    cudaMalloc(&d_a, n * sizeof(float));
    cudaMalloc(&d_b, n * sizeof(float));
    cudaMalloc(&d_c, n * sizeof(float));
    
    // 拷贝数据到设备
    cudaMemcpy(d_a, h_a, n * sizeof(float), cudaMemcpyHostToDevice);
    cudaMemcpy(d_b, h_b, n * sizeof(float), cudaMemcpyHostToDevice);
    
    // 启动 Kernel
    int blockSize = 256;
    int gridSize = (n + blockSize - 1) / blockSize;
    vectorAdd<<<gridSize, blockSize>>>(d_a, d_b, d_c, n);
    
    // 拷贝结果回主机
    cudaMemcpy(h_c, d_c, n * sizeof(float), cudaMemcpyDeviceToHost);
    
    // 释放内存
    cudaFree(d_a);
    cudaFree(d_b);
    cudaFree(d_c);
}
```

### 5.3 设备函数

```cpp
// 设备函数：只能在 GPU 上调用
__device__ float square(float x) {
    return x * x;
}

// 同时可在 Host 和 Device 调用
__host__ __device__ float add(float a, float b) {
    return a + b;
}

// 强制内联
__device__ __forceinline__ float multiply(float a, float b) {
    return a * b;
}
```

---

## 6. CUDA 内存模型

### 6.1 内存类型

```cpp
// 1. Global Memory - 最慢，容量大
__global__ void kernel(float* globalArray) {
    globalArray[threadIdx.x] = 0;  // Global Memory 访问
}

// 2. Shared Memory - 快，Block 内共享
__global__ void kernel() {
    __shared__ float sharedArray[256];  // 静态分配
    sharedArray[threadIdx.x] = 0;
    __syncthreads();  // 同步
}

// 3. Constant Memory - 只读，缓存优化
__constant__ float constArray[256];  // 声明在全局

// 4. Texture Memory - 只读，2D 空间局部性优化
texture<float, 2, cudaReadModeElementType> texRef;

// 5. Local Memory - 私有，但实际在 Global Memory
// 大数组或寄存器溢出时使用，较慢
```

### 6.2 Shared Memory 使用

```cpp
// 矩阵乘法优化示例
__global__ void matrixMul(float* A, float* B, float* C, int N) {
    __shared__ float As[TILE_SIZE][TILE_SIZE];
    __shared__ float Bs[TILE_SIZE][TILE_SIZE];
    
    int tx = threadIdx.x, ty = threadIdx.y;
    int row = blockIdx.y * TILE_SIZE + ty;
    int col = blockIdx.x * TILE_SIZE + tx;
    
    float sum = 0.0f;
    
    for (int t = 0; t < N / TILE_SIZE; t++) {
        // 加载 Tile 到 Shared Memory
        As[ty][tx] = A[row * N + t * TILE_SIZE + tx];
        Bs[ty][tx] = B[(t * TILE_SIZE + ty) * N + col];
        __syncthreads();
        
        // 计算 Tile 内乘法
        for (int k = 0; k < TILE_SIZE; k++) {
            sum += As[ty][k] * Bs[k][tx];
        }
        __syncthreads();
    }
    
    C[row * N + col] = sum;
}
```

### 6.3 统一内存（Unified Memory）

```cpp
// 简化内存管理
float* data;
cudaMallocManaged(&data, n * sizeof(float));  // 统一内存

// Host 和 Device 都可访问
for (int i = 0; i < n; i++) data[i] = i;  // Host 写入
kernel<<<gridSize, blockSize>>>(data);     // Device 使用
cudaDeviceSynchronize();
printf("%f\n", data[0]);                   // Host 读取

cudaFree(data);

// 底层：按需页面迁移
// 优点：编程简单
// 缺点：可能有页面迁移开销
```

---

## 7. 同步与原子操作

### 7.1 同步机制

```cpp
// Block 内同步
__global__ void kernel() {
    __shared__ float s[256];
    s[threadIdx.x] = compute();
    __syncthreads();  // 等待 Block 内所有线程
    use(s[(threadIdx.x + 1) % 256]);
}

// Warp 内同步（隐式）
// 同一 Warp 的线程天然同步执行

// Warp 内显式同步（Volta+）
__syncwarp();

// Grid 级同步 - Cooperative Groups
#include <cooperative_groups.h>
namespace cg = cooperative_groups;

__global__ void kernel() {
    cg::grid_group grid = cg::this_grid();
    // ... 第一阶段
    grid.sync();  // 全 Grid 同步
    // ... 第二阶段
}
```

### 7.2 原子操作

```cpp
// 原子加
atomicAdd(&counter, 1);
atomicAdd(&floatSum, value);  // FP32/FP64 原子加

// 原子比较交换
atomicCAS(&lock, 0, 1);  // if (lock == 0) lock = 1

// 原子最值
atomicMax(&maxVal, newVal);
atomicMin(&minVal, newVal);

// 原子交换
atomicExch(&slot, newValue);

// 示例：直方图
__global__ void histogram(int* data, int* hist, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) {
        atomicAdd(&hist[data[i]], 1);
    }
}
```

---

## 8. CUDA 性能优化

### 8.1 合并访问（Coalesced Access）

```cpp
// Good：连续地址，合并访问
__global__ void good(float* a) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    float val = a[i];  // 线程 0,1,2... 访问 a[0],a[1],a[2]...
}

// Bad：跳跃访问，无法合并
__global__ void bad(float* a, int stride) {
    int i = (blockIdx.x * blockDim.x + threadIdx.x) * stride;
    float val = a[i];  // 线程 0,1,2... 访问 a[0],a[stride],a[2*stride]...
}

// 合并条件：
// - Warp 内线程访问连续地址
// - 地址 128 字节对齐
// - 一次可合并为 1-4 个 128B 事务
```

### 8.2 Bank Conflict

```cpp
// Shared Memory 分为 32 个 Bank
// 每个 Bank 4 字节宽

// Good：无 Bank Conflict
__shared__ float s[32];
float val = s[threadIdx.x];  // 每线程访问不同 Bank

// Bad：Bank Conflict
__shared__ float s[32][32];
float val = s[threadIdx.x][0];  // 所有线程访问同一列（同一 Bank）

// 解决：Padding
__shared__ float s[32][33];  // 每行多一列，错开 Bank
float val = s[threadIdx.x][0];  // 无冲突
```

### 8.3 Occupancy

```cpp
// Occupancy = 活跃 Warp 数 / SM 最大 Warp 数

// 影响因素：
// 1. Block 大小
// 2. 每线程寄存器使用
// 3. 每 Block Shared Memory 使用

// 查询最佳配置
int minGridSize, blockSize;
cudaOccupancyMaxPotentialBlockSize(&minGridSize, &blockSize, kernel, 0, 0);

// Launch Bounds：提示编译器
__global__ void __launch_bounds__(256, 4) kernel() {
    // 256：每 Block 最大线程数
    // 4：每 SM 最小 Block 数
}
```

### 8.4 数据传输优化

```cpp
// Pinned Memory：避免额外拷贝
float* h_data;
cudaMallocHost(&h_data, size);  // Page-locked memory
cudaMemcpy(d_data, h_data, size, cudaMemcpyHostToDevice);  // 更快

// 异步传输
cudaMemcpyAsync(d_data, h_data, size, cudaMemcpyHostToDevice, stream);
kernel<<<grid, block, 0, stream>>>(d_data);
cudaMemcpyAsync(h_result, d_result, size, cudaMemcpyDeviceToHost, stream);

// 多 Stream 重叠
for (int i = 0; i < nStreams; i++) {
    cudaMemcpyAsync(..., stream[i]);
    kernel<<<..., stream[i]>>>();
    cudaMemcpyAsync(..., stream[i]);
}
```

---

## 9. 高级特性

### 9.1 Tensor Core 编程（WMMA）

```cpp
#include <mma.h>
using namespace nvcuda::wmma;

__global__ void wmmaKernel(half* a, half* b, float* c) {
    // 声明 Fragment
    fragment<matrix_a, 16, 16, 16, half, row_major> a_frag;
    fragment<matrix_b, 16, 16, 16, half, col_major> b_frag;
    fragment<accumulator, 16, 16, 16, float> c_frag;
    
    // 初始化累加器
    fill_fragment(c_frag, 0.0f);
    
    // 加载矩阵
    load_matrix_sync(a_frag, a, 16);
    load_matrix_sync(b_frag, b, 16);
    
    // 矩阵乘加
    mma_sync(c_frag, a_frag, b_frag, c_frag);
    
    // 存储结果
    store_matrix_sync(c, c_frag, 16, mem_row_major);
}
```

### 9.2 CUDA Graph

```cpp
// 捕获 Graph
cudaGraph_t graph;
cudaGraphExec_t graphExec;

cudaStreamBeginCapture(stream, cudaStreamCaptureModeGlobal);
kernel1<<<...>>>();
kernel2<<<...>>>();
kernel3<<<...>>>();
cudaStreamEndCapture(stream, &graph);

cudaGraphInstantiate(&graphExec, graph, NULL, NULL, 0);

// 执行（减少启动开销）
for (int i = 0; i < iterations; i++) {
    cudaGraphLaunch(graphExec, stream);
}
```

### 9.3 Multi-GPU

```cpp
// P2P 访问
cudaSetDevice(0);
cudaDeviceEnablePeerAccess(1, 0);  // 允许 GPU 0 访问 GPU 1 内存

// NCCL 集合通信
ncclComm_t comm;
ncclCommInitRank(&comm, nGPUs, id, rank);

ncclAllReduce(sendbuff, recvbuff, count, ncclFloat, ncclSum, comm, stream);
```

---

## 10. 面试题自查

**Q1: GPU 和 CPU 在设计上有什么根本区别？**

CPU 优化延迟（少量强大核心、大缓存、复杂控制），GPU 优化吞吐（大量简单核心、高带宽、简单控制）。CPU 适合串行逻辑，GPU 适合大规模并行计算。

**Q2: 什么是 Warp？Warp Divergence 是什么？**

Warp 是 32 个线程的集合，是 GPU 调度的基本单位，执行 SIMT（同一指令多线程）。Divergence 是同一 Warp 内线程走不同分支，必须串行执行所有分支路径，降低效率。

**Q3: CUDA 的线程层次是什么？**

Grid → Block → Thread。Grid 是 Kernel 的整个执行实例，Block 是可协作的线程组（共享 Shared Memory），Thread 是最小执行单元。通过 threadIdx/blockIdx/blockDim/gridDim 访问。

**Q4: Shared Memory 有什么用？Bank Conflict 是什么？**

Shared Memory 是 Block 内线程共享的高速内存（~30 周期），用于线程间通信和数据复用。它分为 32 个 Bank，多线程访问同一 Bank 时产生 Conflict，需串行访问。解决方法：Padding。

**Q5: 什么是合并访问（Coalesced Access）？**

当 Warp 内线程访问连续且对齐的内存地址时，多个访问合并为一次内存事务，大幅提高带宽利用率。非合并访问会产生多个事务，降低性能。

**Q6: Tensor Core 是什么？和 CUDA Core 有什么区别？**

Tensor Core 是矩阵乘加加速单元，一个周期可完成小矩阵（如 16×16×16）运算；CUDA Core 是标量 ALU，一个周期完成一次乘加。Tensor Core 用于深度学习，吞吐量高数十倍。

**Q7: GPU 内存层次从快到慢是什么？**

Register（~1 周期）→ Shared Memory（~30 周期）→ L1/L2 Cache → Global Memory/HBM（~400 周期）。优化原则：尽量使用高层次内存，减少 Global Memory 访问。

**Q8: CUDA Stream 的作用是什么？**

Stream 是异步操作队列，同一 Stream 内操作串行，不同 Stream 可并行。用于重叠计算与传输、多 Kernel 并发执行，隐藏延迟。

**Q9: 什么是 Occupancy？如何优化？**

Occupancy = 活跃 Warp 数 / SM 最大 Warp 数。受 Block 大小、寄存器用量、Shared Memory 用量限制。使用 cudaOccupancyMaxPotentialBlockSize 查询最佳配置。

**Q10: cudaMallocManaged 和 cudaMalloc 的区别？**

cudaMallocManaged 分配统一内存，Host 和 Device 都可直接访问，系统自动迁移页面，编程简单但可能有迁移开销；cudaMalloc 只分配设备内存，需显式拷贝。

**Q11: CUDA Graph 有什么用？**

将一系列 CUDA 操作（Kernel、拷贝）捕获为图，然后一次性提交执行，减少 CPU 端启动开销。适合重复执行相同工作流的场景。

**Q12: NVLink 和 PCIe 有什么区别？**

NVLink 是 NVIDIA 专有高速互联（NVLink 4.0: 900GB/s 双向），用于 GPU 间和 GPU-CPU 互联；PCIe 5.0 约 64GB/s。NVLink 带宽高 10+ 倍，用于多 GPU 训练。

**Q13: 什么是 NCCL？**

NVIDIA Collective Communications Library，用于多 GPU 集合通信（AllReduce、AllGather 等），优化了 NVLink/InfiniBand 拓扑，是分布式训练的基础。

**Q14: 如何选择 Block 大小？**

通常选择 128/256/512，保证是 32 的倍数（Warp 大小）。考虑 Occupancy、Shared Memory 需求、寄存器压力。使用 Nsight Compute 分析具体情况。

**Q15: __syncthreads() 和 atomicAdd 的区别？**

__syncthreads() 是 Block 内线程同步屏障，所有线程必须到达才能继续；atomicAdd 是原子操作，保证单个内存位置的读-改-写操作不被打断，用于多线程安全更新同一变量。
