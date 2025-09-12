## 基本单元
```
inline torch::nn::Conv2dOptions conv_options(int64_t in_planes, int64_t out_planes, int64_t kerner_size,
    int64_t stride = 1, int64_t padding = 0, bool with_bias = false) {
    torch::nn::Conv2dOptions conv_options = torch::nn::Conv2dOptions(in_planes, out_planes, kerner_size);
    conv_options.stride(stride);
    conv_options.padding(padding);
    conv_options.bias(with_bias);
    return conv_options;
}
```
```
class ConvReluBnImpl : public torch::nn::Module {
public:
    ConvReluBnImpl(int input_channel=3, int output_channel=64, int kernel_size = 3, int stride = 1);
    torch::Tensor forward(torch::Tensor x);
private:
    // Declare layers
    torch::nn::Conv2d conv{ nullptr };
    torch::nn::BatchNorm2d bn{ nullptr };
};
TORCH_MODULE(ConvReluBn);

ConvReluBnImpl::ConvReluBnImpl(int input_channel, int output_channel, int kernel_size, int stride) {
    conv = register_module("conv", torch::nn::Conv2d(conv_options(input_channel,output_channel,kernel_size,stride,kernel_size/2)));
    bn = register_module("bn", torch::nn::BatchNorm2d(output_channel));

}

torch::Tensor ConvReluBnImpl::forward(torch::Tensor x) {
    x = torch::relu(conv->forward(x));
    x = bn(x);
    return x;
}
```
## 例子
```
class plainCNN : public torch::nn::Module{
public:
    plainCNN(int in_channels, int out_channels);
    torch::Tensor forward(torch::Tensor x);
private:
    int mid_channels[3] = {32,64,128};
    ConvReluBn conv1{nullptr};
    ConvReluBn down1{nullptr};
    ConvReluBn conv2{nullptr};
    ConvReluBn down2{nullptr};
    ConvReluBn conv3{nullptr};
    ConvReluBn down3{nullptr};
    torch::nn::Conv2d out_conv{nullptr};
};

plainCNN::plainCNN(int in_channels, int out_channels){
    conv1 = ConvReluBn(in_channels,mid_channels[0],3);
    down1 = ConvReluBn(mid_channels[0],mid_channels[0],3,2);
    conv2 = ConvReluBn(mid_channels[0],mid_channels[1],3);
    down2 = ConvReluBn(mid_channels[1],mid_channels[1],3,2);
    conv3 = ConvReluBn(mid_channels[1],mid_channels[2],3);
    down3 = ConvReluBn(mid_channels[2],mid_channels[2],3,2);
    out_conv = torch::nn::Conv2d(conv_options(mid_channels[2],out_channels,3));

    conv1 = register_module("conv1",conv1);
    down1 = register_module("down1",down1);
    conv2 = register_module("conv2",conv2);
    down2 = register_module("down2",down2);
    conv3 = register_module("conv3",conv3);
    down3 = register_module("down3",down3);
    out_conv = register_module("out_conv",out_conv);
}

torch::Tensor plainCNN::forward(torch::Tensor x){
    x = conv1->forward(x);
    x = down1->forward(x);
    x = conv2->forward(x);
    x = down2->forward(x);
    x = conv3->forward(x);
    x = down3->forward(x);
    x = out_conv->forward(x);
    return x;
}
```