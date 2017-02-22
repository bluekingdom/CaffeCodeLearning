# Blob

blob定义于blob.hpp/cpp文件中。它是Caffe基本的运算单元，用于Layers之间的数据交互。基本上它是对[SyncedMemory](note/basics/syncedmem.md)进行封装，定义成一个可以表示多维数据的一个结构。

## member variables

```c++
shared_ptr<SyncedMemory> data_; // 存储前向传播用到的数据
shared_ptr<SyncedMemory> diff_; // 存储反向传播用到的数据，一般表示为误差项
shared_ptr<SyncedMemory> shape_data_; // 表示 data_, diff_ 的维度，和shape_是相同的。
vector<int> shape_; // 同样是数据维度，对于图片输入，一般是[batch_size, channel, height, width] 这4维。
int count_; // data_, diff_ 存储的实际数据的长度，也就是所有shape_的元素相乘
int capacity_; // data_, diff_ 分配的内存大小，会和count_不同
```

成员变量比较少，也容易理解。要注意的是，count_ 肯定是等于  $\prod_{i=0}^{shape.size()}shape [i]$，而capacity_ 只d表示data、diff分配的内存大小，不一定和count_ 相等， 这在Reshape()函数中会找到原因。

## member functions

Blob类的函数大多是比较直观，没有比较复杂的逻辑。有一些函数需要注意一下：

* `void Blob<Dtype>::Reshape(const vector<int>& shape)`:  根据输入的数据维度，改变对应的成员变量。注意到

```C++
if (count_ > capacity_) { // 如果新的维度大于之前的内存大小
  capacity_ = count_;
  data_.reset(new SyncedMemory(capacity_ * sizeof(Dtype)));
  diff_.reset(new SyncedMemory(capacity_ * sizeof(Dtype)));
}
```

如果新的维度小于之前分配的内存大小，那将继续使用之前的内存空间。

* `void Update()`: 其中用到`void caffe_axpy(const int N, const Dtype alpha, const Dtype* X, Dtype* Y)` 这个函数，定义在math_function.hpp中，是caffe提供的一个数学库。caffe_axpy的运算是$Y = alpha * X + Y$, 所以Update函数的功能就是$data_i = data_i - diff_i$. 


* `int offset(int n, int c, int h, int w), int offset(const vector<int>& indices)`: 返回指定维度下的数据偏移量。用于读取某个数据，如`data_at(),diff_at()`这两个函数。
* `asum_data(), asum_diff()` :  对每个元素的绝对值求和。公式是$\sum |d_i|$。
* `sumsq_data(), sumsq_diff()`: 对每个元素的平方值求和，再开根号。公式是$\sqrt{\sum d_i^2}$。
* `ShapeData(const Blob& other), ShapeDiff(const Blob& other)`: 接受other传入的blob，在使用`shared_ptr<SyncedMemory>::reset()`的时候，会优先调用SyncedMemory的析构函数，再进行赋值。
* `FromProto(), ToProto()`: Blob数据的序列化函数，读取和写入到caffemodel文件里面。注意`FromPtoto`优先读取double类型的数据，当没有double类型的数据时，才读取float类型数据。

## 难点 ：

* 关于shape_data_的作用，为什么要额外用SyncedMemory类来定义： TODO