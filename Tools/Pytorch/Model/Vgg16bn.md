## .h
```
//和前面章节一致，定义一个确定conv超参数的函数
inline torch::nn::Conv2dOptions conv_options(int64_t in_planes, int64_t out_planes, int64_t kerner_size,
    int64_t stride = 1, int64_t padding = 0, bool with_bias = false) {
    torch::nn::Conv2dOptions conv_options = torch::nn::Conv2dOptions(in_planes, out_planes, kerner_size);
    conv_options.stride(stride);
    conv_options.padding(padding);
    conv_options.bias(with_bias);
    return conv_options;
}

//仿照上面的conv_options，定义一个确定MaxPool2d的超参数的函数
inline torch::nn::MaxPool2dOptions maxpool_options(int kernel_size, int stride){
    torch::nn::MaxPool2dOptions maxpool_options(kernel_size);
    maxpool_options.stride(stride);
    return maxpool_options;
}

//对应pytorch中的make_features函数，返回CNN主体，该主体是一个torch::nn::Sequential对象
torch::nn::Sequential make_features(vector<int> &cfg, bool batch_norm);

//VGG类的声明，包括初始化和前向传播
class VGGImpl: public torch::nn::Module
{
private:
    torch::nn::Sequential features_{nullptr};
    torch::nn::AdaptiveAvgPool2d avgpool{nullptr};
    torch::nn::Sequential classifier;
public:
    VGGImpl(vector<int> &cfg, int num_classes = 1000, bool batch_norm = false);
    torch::Tensor forward(torch::Tensor x);
};
TORCH_MODULE(VGG);
//vgg16bn函数的声明
VGG vgg16bn(int num_classes);
```
## .cpp
```
torch::nn::Sequential make_features(vector<int> &cfg, bool batch_norm){
    torch::nn::Sequential features;
    int in_channels = 3;
    for(auto v : cfg){
        if(v==-1){
            features->push_back(torch::nn::MaxPool2d(maxpool_options(2,2)));
        }
        else{
            auto conv2d = torch::nn::Conv2d(conv_options(in_channels,v,3,1,1));
            features->push_back(conv2d);
            if(batch_norm){
                features->push_back(torch::nn::BatchNorm2d(torch::nn::BatchNorm2dOptions(v)));
            }
            features->push_back(torch::nn::ReLU(torch::nn::ReLUOptions(true)));
            in_channels = v;
        }
    }
    return features;
}

VGGImpl::VGGImpl(vector<int> &cfg, int num_classes, bool batch_norm){
    features_ = make_features(cfg,batch_norm);
    avgpool = torch::nn::AdaptiveAvgPool2d(torch::nn::AdaptiveAvgPool2dOptions(7));
    classifier->push_back(torch::nn::Linear(torch::nn::LinearOptions(512 * 7 * 7, 4096)));
    classifier->push_back(torch::nn::ReLU(torch::nn::ReLUOptions(true)));
    classifier->push_back(torch::nn::Dropout());
    classifier->push_back(torch::nn::Linear(torch::nn::LinearOptions(4096, 4096)));
    classifier->push_back(torch::nn::ReLU(torch::nn::ReLUOptions(true)));
    classifier->push_back(torch::nn::Dropout());
    classifier->push_back(torch::nn::Linear(torch::nn::LinearOptions(4096, num_classes)));

    features_ = register_module("features",features_);
    classifier = register_module("classifier",classifier);
}

torch::Tensor VGGImpl::forward(torch::Tensor x){
    x = features_->forward(x);
    x = avgpool(x);
    x = torch::flatten(x,1);
    x = classifier->forward(x);
    return torch::log_softmax(x, 1);
}

VGG vgg16bn(int num_classes){
    vector<int> cfg_dd = {64, 64, -1, 128, 128, -1, 256, 256, 256, -1, 512, 512, 512, -1, 512, 512, 512, -1};
    VGG vgg = VGG(cfg_dd,num_classes,true);
    return vgg;
}
```
## 保存模型
不能直接用torch.save，这样存下来的模型不能被 c++ 加载，利用部署时常用的t orch.jit.script 模型来保存
```
import torch
from torchvision.models import vgg16,vgg16_bn
model=model.to(torch.device("cpu"))
model.eval()
var=torch.ones((1,3,224,224))
traced_script_module = torch.jit.trace(model, var)
traced_script_module.save("vgg16bn.pt")
```
## 加载模型
```
vector<int> cfg_16bn = {64, 64, -1, 128, 128, -1, 256, 256, 256, -1, 512, 512, 512, -1, 512, 512, 512, -1};
auto vgg16bn = VGG(cfg_16bn,1000,true);
torch::load(vgg16bn,"your path to vgg16bn.pt");
```
## 初始化
```
class Classifier{
private:
    torch::Device device = torch::Device(torch::kCPU);
    VGG vgg = VGG{nullptr};
public:
    Classifier(int gpu_id = 0);
    void Initialize(int num_classes, string pretrained_path);
    void Train(int epochs, int batch_size, float learning_rate, string train_val_dir, string image_type, string save_path);
    int Predict(cv::Mat &image);
    void LoadWeight(string weight);
};
Classifier::Classifier(int gpu_id)
{
    if (gpu_id >= 0) {
        device = torch::Device(torch::kCUDA, gpu_id);
    }
    else {
        device = torch::Device(torch::kCPU);
    }
}
void Classifier::LoadWeight(string weight){
    torch::load(vgg,weight);
    vgg->eval();
    return;
}
void Classifier::Initialize(int _num_classes, string _pretrained_path){
    vector<int> cfg_d = {64, 64, -1, 128, 128, -1, 256, 256, 256, -1, 512, 512, 512, -1, 512, 512, 512, -1};
    auto net_pretrained = VGG(cfg_d,1000,true);
    vgg = VGG(cfg_d,_num_classes,true);
    torch::load(net_pretrained, _pretrained_path);
    torch::OrderedDict<string, at::Tensor> pretrained_dict = net_pretrained->named_parameters();
    torch::OrderedDict<string, at::Tensor> model_dict = vgg->named_parameters();

    for (auto n = pretrained_dict.begin(); n != pretrained_dict.end(); n++){
        if (strstr((*n).key().data(), "classifier")) {
            continue;
        }
        model_dict[(*n).key()] = (*n).value();
    }
    torch::autograd::GradMode::set_enabled(false);  // 使参数可以拷贝
    auto new_params = model_dict; 
    auto params = vgg->named_parameters(true );
    auto buffers = vgg->named_buffers(true);
    for (auto& val : new_params) {
        auto name = val.key();
        auto* t = params.find(name);
        if (t != nullptr) {
            t->copy_(val.value());
        }
        else {
            t = buffers.find(name);
            if (t != nullptr) {
                t->copy_(val.value());
            }
        }
    }
    torch::autograd::GradMode::set_enabled(true);
    try{
        vgg->to(device);
    }
    catch (const exception&e)    {
        cout << e.what() << endl;
    }
    return;
}
```
## 训练
```
void Classifier::Train(int num_epochs, int batch_size, float learning_rate, string train_val_dir, string image_type, string save_path){
    string path_train = train_val_dir+ "\\train";
    string path_val = train_val_dir + "\\val";

    auto custom_dataset_train = dataSetClc(path_train, image_type).map(torch::data::transforms::Stack<>());
    auto custom_dataset_val = dataSetClc(path_val, image_type).map(torch::data::transforms::Stack<>());

    auto data_loader_train = torch::data::make_data_loader<torch::data::samplers::RandomSampler>(move(custom_dataset_train), batch_size);
    auto data_loader_val = torch::data::make_data_loader<torch::data::samplers::RandomSampler>(move(custom_dataset_val), batch_size);

    float loss_train = 0; float loss_val = 0;
    float acc_train = 0.0; float acc_val = 0.0; float best_acc = 0.0;
    for (size_t epoch = 1; epoch <= num_epochs; ++epoch) {
        size_t batch_index_train = 0;
        size_t batch_index_val = 0;
        if (epoch == int(num_epochs / 2)) { learning_rate /= 10; }
        torch::optim::Adam optimizer(vgg->parameters(), learning_rate); // 学习率
        if (epoch < int(num_epochs / 8)){
            for (auto mm : vgg->named_parameters()){
                if (strstr(mm.key().data(), "classifier"))
                {
                    mm.value().set_requires_grad(true);
                }
                else
                {
                    mm.value().set_requires_grad(false);
                }
            }
        }
        else {
            for (auto mm : vgg->named_parameters())
            {
                mm.value().set_requires_grad(true);
            }
        }
        // 遍历data_loader，产生批次
        for (auto& batch : *data_loader_train) {
            auto data = batch.data;
            auto target = batch.target.squeeze();
            data = data.to(torch::kF32).to(device).div(255.0);
            target = target.to(torch::kInt64).to(device);
            optimizer.zero_grad();
            // Execute the model
            torch::Tensor prediction = vgg->forward(data);
            auto acc = prediction.argmax(1).eq(target).sum();
            acc_train += acc.template item<float>() / batch_size;
            // 计算损失大小
            torch::Tensor loss = torch::nll_loss(prediction, target);
            // 计算梯度
            loss.backward();
            // 更新权重
            optimizer.step();
            loss_train += loss.item<float>();
            batch_index_train++;
            cout << "Epoch: " << epoch << " |Train Loss: " << loss_train / batch_index_train << " |Train Acc:" << acc_train / batch_index_train << "\r";
        }
        cout << endl;
        //验证
        vgg->eval();
        for (auto& batch : *data_loader_val) {
            auto data = batch.data;
            auto target = batch.target.squeeze();
            data = data.to(torch::kF32).to(device).div(255.0);
            target = target.to(torch::kInt64).to(device);
            torch::Tensor prediction = vgg->forward(data);
            // 计算损失,NLL和Log_softmax配合形成交叉熵损失
            torch::Tensor loss = torch::nll_loss(prediction, target);
            auto acc = prediction.argmax(1).eq(target).sum();
            acc_val += acc.template item<float>() / batch_size;
            loss_val += loss.item<float>();
            batch_index_val++;
            cout << "Epoch: " << epoch << " |Val Loss: " << loss_val / batch_index_val << " |Valid Acc:" << acc_val / batch_index_val << "\r";
        }
        cout << endl;
        if (acc_val > best_acc) {
            torch::save(vgg, save_path);
            best_acc = acc_val;
        }
        loss_train = 0; loss_val = 0; acc_train = 0; acc_val = 0; batch_index_train = 0; batch_index_val = 0;
    }
}
```
## 预测
```
int Classifier::Predict(cv::Mat& image){
    cv::resize(image, image, cv::Size(448, 448));
    torch::Tensor img_tensor = torch::from_blob(image.data, { image.rows, image.cols, 3 }, torch::kByte).permute({ 2, 0, 1 });
    img_tensor = img_tensor.to(device).unsqueeze(0).to(torch::kF32).div(255.0);
    auto prediction = vgg->forward(img_tensor);
    prediction = torch::softmax(prediction,1);
    auto class_id = prediction.argmax(1);
    int ans = int(class_id.item().toInt());
    float prob = prediction[0][ans].item().toFloat();
    return ans;
}
```