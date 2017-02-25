# Layer

该类定义在layer.hpp/cpp中，是所有Layer的基类。它的主要功能是进行前向后向（Forward & Backward）的计算，以及为这些计算进行一些必要的设置。同时它为派生类提供了一些初始化、前向后向计算的接口。

## member variables

该类的注释十分详尽，成员变量基本上都比较好理解了。

```C++
LayerParameter layer_param_; // 定义Layer的结构体，结构体的定义在caffe.proto中可以查看
vector<shared_ptr<Blob<Dtype> > > blobs_; // Layer的权重(weights)
vector<bool> param_propagate_down_; // 是否计算各个Layer Weights的误差项(diff)
vector<Dtype> loss_; // 存储各个top blob的Loss Weights
bool is_shared_; // 该是否和其他Net进行共享的
shared_ptr<boost::mutex> forward_mutex_; // 前向计算的线程同步锁
```

## member functions

* `Layer`构造函数：Layer对象创建时，接收一个LayerParameter参数。通过这个参数，进行Layer的初始化工作。

```C++
explicit Layer(const LayerParameter& param)
  : layer_param_(param), is_shared_(false) {
    phase_ = param.phase(); // Train or Test
    if (layer_param_.blobs_size() > 0) { // 初始化Layer的weight
      blobs_.resize(layer_param_.blobs_size());
      for (int i = 0; i < layer_param_.blobs_size(); ++i) {
        blobs_[i].reset(new Blob<Dtype>());
        blobs_[i]->FromProto(layer_param_.blobs(i)); // 设置Blob的维度，如果param中有数据就复制数据
      }    }  }
```

* `SetUp`函数：该函数根据构造函数传入的layer_param_参数，初始化Layer的一些参数，检查bottom和top的向量的大小，并设置top blob的维度。

```C++
void SetUp(const vector<Blob<Dtype>*>& bottom, const vector<Blob<Dtype>*>& top) {
  InitMutex(); // 初始化同步锁
  CheckBlobCounts(bottom, top); // 检查bottom,top的个数
  LayerSetUp(bottom, top); // virtual函数，在各自Layer会继承，进行一些初始化工作
  Reshape(bottom, top); // abstract函数，必须在各自Layer继承，设置top blob的维度
  SetLossWeights(top); // 设置loss weight，如果该Layer有的话
}
```

* `SetLossWeights`函数：设置loss weight，其中注意到，loss weight会被赋值到top blob的diff中。

```C++
Dtype* loss_multiplier = top[top_id]->mutable_cpu_diff();
caffe_set(count, loss_weight, loss_multiplier); // 数组赋值
```

* `Forward`函数：前向计算的wrapper函数，调用各个Layer实现的`Forward_cpu(gpu)`函数。

```C++
Reshape(bottom, top); // 重新调整top blob
switch (Caffe::mode()) {
case Caffe::CPU:
  Forward_cpu(bottom, top); // cpu的前向计算
  for (int top_id = 0; top_id < top.size(); ++top_id) {
    if (!this->loss(top_id)) { continue; } // 如果是设置了loss weight
    const int count = top[top_id]->count();
    const Dtype* data = top[top_id]->cpu_data();
    const Dtype* loss_weights = top[top_id]->cpu_diff(); // loss weight 存放在 top diff
    loss += caffe_cpu_dot(count, data, loss_weights); // 计算该top贡献的loss，累加到total loss中
  }
  break;
case Caffe::GPU: // gpu流程和cpu一致
  Forward_gpu(bottom, top);
  ...
```



## 难点

* Forward计算时，`bottom, layer weight, top`这三个Blob的计算关系： 一般是通过bottom.data 和 weight.data 的计算，得到top.data。


* Backward计算时，`bottom, layer weight, top`这三个Blob的计算关系：一般是通过 top.diff、bottom.data等得到weight.diff 和 bottom.diff。weight.diff用于Layer的权值更新（也就是weight.data的更新），而bottom.diff用于误差项的传导（也就是计算其他Layer的top.diff）。
* `bool is_shared_`参数为`true`时的意义，什么时候会使用`is_shared_==true`：TODO

