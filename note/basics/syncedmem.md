# SyncedMemory

SyncedMemory定义于syncedmem.hpp/cpp文件中。用于管理数据的内存分配，以及数据在内存和显存中的一致性。

首先来看该类中重要的成员变量：

```C++
void* cpu_ptr_; // 数据在内存中的指针
void* gpu_ptr_; // 数据在显存中的指针
size_t size_; // 数据长度
enum SyncedHead { // 数据同步状态 
  UNINITIALIZED, // 该数据没初始化
  HEAD_AT_CPU, // 内存数据为最新
  HEAD_AT_GPU, // 显存数据为最新
  SYNCED // 内存和显存的数据为一致
};
SyncedHead head_; // 记录当前同步状态
bool own_cpu_data_; // 内存是否由自己分配的
bool own_gpu_data_; // 显存是否由自己分配的
bool cpu_malloc_use_cuda_; // 是否是使用cuda函数来分配内存
```

熟悉这几个变量之后，以下函数就十分好理解了：

* `to_cpu()`：将最新的数据同步到内存中。
* `to_gpu()` : 将最新的数据同步到显存中。
* `cpu_data(), gpu_data(), mutable_cpu_data(), mutable_gpu_data `: 获取 内存\显存 数据的指针。`xpu_data()`函数返回的是`const void*`型指针， 以表示数据是只读的。而带*mutable*的函数返回值没有`const`，且多加了一行`head_ = HEAD_AT_XPU`，表明返回指针所存储的数据是最新的。
* `set_cpu_data(), set_gpu_data()` : 接受外部数据的指针，将自己的数据指针指向外部数据。该函数会释放自己原来分配的数据，并将`own_cpu_data,own_gpu_data`变量设为`false`。

有以下几个难点：

* 数据有三种方式分配：`malloc, cudaMallocHost, cudaMalloc`。 前两个是分配内存数据的，最后一个是分配显存数据。两个内存分配方式的差别在于和显存进行数据传输的速度差异，在传输数据大小大于一定条件下，`cudaMallocHost`会比`malloc`的要快。具体请点击阅读[Pinned-Memory-Vs-Non-Pinned-Memory](https://satisfie.github.io/2016/09/15/Pinned-Memory-Vs-Non-Pinned-Memory/)。
* `async_gpu_push`函数： 异步执行将内存数据同步到显存。 用到`cudaMemcpyAsync`函数，以及base_data_layer.cpp中的`cudaStreamSynchronize`，完成异步传输工作。 这里还设计cuda stream的概念，详情点击阅读[CUDA 流](http://www.voidcn.com/blog/u013947807/article/p-4917081.html)和[cudaMemcpy与cudaMemcpyAsync的区别](http://www.cnblogs.com/shrimp-can/p/5231857.html)。 

