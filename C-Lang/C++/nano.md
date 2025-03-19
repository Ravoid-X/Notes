# GPIO
1. 克隆存储库
   ````
   git clone https://github.com/pjueon/JetsonGPIO
   ````
2. 添加到CMakeLists.txt
3. 四种I/O 引脚编号方法
   ````
   GPIO::setmode(GPIO::BOARD); 、、
   GPIO::setmode(GPIO::BCM);
   GPIO::setmode(GPIO::CVM);
   GPIO::setmode(GPIO::TEGRA_SOC);
   ````
4. 检查模式
   ````
   GPIO::NumberingModes mode = GPIO::getmode();
   ````
5. 引脚设置
   ````
   GPIO::setup(channel, GPIO::IN); // channel must be int or std::string
   GPIO::setup(channel, GPIO::OUT);
   ````
6. 输入
   ````
   int value = GPIO::input(channel);
   ````
7. 输出
   ````
   GPIO::output(channel, state);
   ````
8. 清理设置（程序结束时）
   ````
   GPIO::cleanup();
   GPIO::cleanup(chan1); // cleanup only chan1
   ````
9. 中断
   ````
   GPIO::wait_for_edge(channel, GPIO::RISING/FALLING/BOTH)
   ````
   此函数会阻塞调用线程，直到检测到提供的边
   ````
   GPIO::WaitResult result = GPIO::wait_for_edge(channel, GPIO::RISING, 10, 500);
   ````
   返回一个GPIO::WaitResult对象，其中包含检测到边缘的通道名称