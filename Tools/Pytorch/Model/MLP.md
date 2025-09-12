## 基本单元
```
class LinearBnReluImpl : public torch::nn::Module{
public:
    LinearBnReluImpl(int input_features, int output_features);
    torch::Tensor forward(torch::Tensor x);
private:
    torch::nn::Linear ln{nullptr};
    torch::nn::BatchNorm1d bn{nullptr};
};TORCH_MODULE(LinearBnRelu);

LinearBnReluImpl::LinearBnReluImpl(int in_features, int out_features){
    ln = register_module("ln", torch::nn::Linear(torch::nn::LinearOptions(in_features, out_features)));
    bn = register_module("bn", torch::nn::BatchNorm1d(out_features));
}

torch::Tensor LinearBnReluImpl::forward(torch::Tensor x){
    x = torch::relu(ln->forward(x));
    x = bn(x);
    return x;
}
```
1. 自定义的神经网络模块必须继承自torch::nn::Module，提供了必要的功能，如参数注册、子模块管理、模型保存/加载等。
2. ln->forward(x)，输入进行矩阵乘法和偏置加法
## 例子
```
class MLP: public torch::nn::Module{
public:
    MLP(int in_features, int out_features);
    torch::Tensor forward(torch::Tensor x);
private:
    int mid_features[3] = {32,64,128};//三个隐藏层各自的输出特征数
    LinearBnRelu ln1{nullptr};
    LinearBnRelu ln2{nullptr};
    LinearBnRelu ln3{nullptr};
    torch::nn::Linear out_ln{nullptr};//输出层
};

MLP::MLP(int in_features, int out_features){
    ln1 = LinearBnRelu(in_features, mid_features[0]);
    ln2 = LinearBnRelu(mid_features[0], mid_features[1]);
    ln3 = LinearBnRelu(mid_features[1], mid_features[2]);
    out_ln = torch::nn::Linear(mid_features[2], out_features);

    ln1 = register_module("ln1", ln1);
    ln2 = register_module("ln2", ln2);
    ln3 = register_module("ln3", ln3);
    out_ln = register_module("out_ln",out_ln);
}
torch::Tensor MLP::forward(torch::Tensor x){
    x = ln1->forward(x);
    x = ln2->forward(x);
    x = ln3->forward(x);
    x = out_ln->forward(x);
    return x;
}
```