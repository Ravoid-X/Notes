# Windows
官网下载相应版本\
https://opencv.org/releases/
## MinGW编译
1. 打开 CMake-GUI，设置路径如下\
<img src="../../pic/CV/OpenCV/mingw-build-path.png" style="width:500px;padding:10px;"/>

2. 点击 configure\
<img src="../../pic/CV/OpenCV/mingw-build-configure.png" style="width:500px;padding:10px;"/> \
如果报找不到编译器的错误,可以选择第二个单选框 Specify native compilers 手动选择编译器路径

3. 配置后再次点击 configure\
（1）填入 Release 会编译发行版本的 opencv 包，从而去除 debug 信息和符号表，这可以提高性能；填入 Debug 则会编译 debug 版本的 opencv，适合需要修改 opencv 源码的情况\
<img src="../../pic/CV/OpenCV/mingw-build-configure1.png" style="width:500px;padding:10px;"/> \
（2）会使得生成的链接库为一个包而不是多个\
<img src="../../pic/CV/OpenCV/mingw-build-configure2.png" style="width:500px;padding:10px;"/> \
（3）会生成一个 pkg-config 的路径使得 pkgconfig 能够自动传递库路径给 g++ 进行编译\
<img src="../../pic/CV/OpenCV/mingw-build-configure3.png" style="width:500px;padding:10px;"/> 

4. 点击 Generate
5. 到 build 路径下运行 ``make``，完成后运行 ``make install``
6. 将 ``C:\ruanjian\opencv\mingw64-build\install\x64\mingw\bin``添加到系统变量路径中
7. CMakeLists添加
```
set(OpenCV_DIR  C:/ruanjian/opencv/build/install)
find_package(OpenCV REQUIRED)
include_directories(${OpenCV_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME} ${OpenCV_LIBS})
```
## MVSC编译
1. 将 ``C:\ruanjian\opencv\build\x64\vc16\bin``添加到系统变量路径中
2. CMakeLists添加
```
set(OpenCV_DIR  C:/ruanjian/opencv/build/x64/vc16/lib/)
find_package(OpenCV REQUIRED)
include_directories(${OpenCV_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME} ${OpenCV_LIBS})
```
# Ubuntu
## 下载
`wget https://github.com/opencv/opencv/archive/refs/tags/4.12.0.zip`
## 安装依赖
```
sudo apt-get install build-essential
sudo apt-get install cmake-gui git libgtk2.0-dev pkg-config libavcodec-dev libavformat-dev libswscale-dev
sudo apt-get install python3-dev python3-numpy libtbb2 libtbb-dev libjpeg-dev libpng-dev
```
## 编译
```
mkdir -p build && cd build
cmake ../../opencv412 ..
make -j16
sudo make install
```
## 配置
安装路径 `/usr/local/include/opencv4`；库路径 `/usr/local/lib`
### 添加库路径
1. `sudo gedit /etc/ld.so.conf.d/opencv.conf`
2. 添加 `/usr/local/lib`
### 配置安装路径
1. 
```
sudo ldconfig
sudo gedit /etc/bash.bashrc
```
2. 末尾添加
```
PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/local/opencv4/lib/pkgconfig 
export PKG_CONFIG_PATH
```
3. 更新环境变量\
`source /etc/bash.bashrc`
## 测试
```
python3
import cv2
cv2.__version__
```