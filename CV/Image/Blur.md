## 原因
在相机的“快门开启”时间内，物体移动了较长距离，导致其影像在传感器上拖出一条轨迹。
## 检测
1. 计算拉普拉斯算子响应的方差，其对图像中的边缘和噪声非常敏感。
2. 清晰的图像具有锐利的边缘，经过拉普拉斯算子处理后，会产生大量的高响应值。因此，整张图的像素值方差会很大。
### 代码
```
double calculateBlurScore(const cv::Mat& image) {
    cv::Mat gray, laplacian;
    cv::cvtColor(image, gray, cv::COLOR_BGR2GRAY); // 转换为灰度图
    cv::Laplacian(gray, laplacian, CV_64F);       // 计算拉普拉斯算子

    cv::Scalar mean, stddev;
    cv::meanStdDev(laplacian, mean, stddev);      // 计算标准差
    return stddev.val[0] * stddev.val[0];         // 返回方差
}
```
## 修复
### 盲反卷积

### 维纳滤波
