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
```

理解这几个变量之后，我们来看该类的一些重要函数：

`to_cpu()`, `to_gpu()`: 