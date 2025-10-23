## 定义
从用户终端（如手机、电脑、平板、网络电视等）到运营商城域网之间的所有通信设备组成的网络，传输距离一般为几百米到几公里
<img src="../../Pic/Subject/Network/access-network.png" style="width:400px;padding:10px;"/>

## 分类
### 有线
1. 铜缆：使用xDSL（x Digital Subscriber Line，x数字用户线）技术，如过去使用电话线拨号上网。
2. 光纤同轴混合：一种灵活的混合使用光纤和同轴电缆的技术，如家里的有线电视。
3. 光纤：使用全光纤接入的PON（Passive Optical Network，无源光网络）技术，是目前有线接入网的主流技术，FTTH（Fiber to the Home，光纤到户）带来超高网速。
### 无线
1. 固定无线：服务的是固定位置的用户或小范围移动的用户，主要技术包括蓝牙、Wi-Fi、WiMAX等。
2. 移动无线：服 务的是大量使用移动终端（如手机）的用户，主要技术是蜂窝移动技术4G、5G等。
## PON
向上通过城域网连接到各种服务提供商的网络（如互联网、IPTV、电话/视频服务等），向下将用户家里的电话、IPTV电视机、电脑等终端连接起来并提供网络服务。
<img src="../../Pic/Subject/Network/access-pon.png" style="width:400px;padding:10px;"/>

1. ONU（Optical Network Unit，光网络单元）：一般直接安装部署到用户家里，常见的有SFU（Single family Unit，单家庭用户单元）和HGU（Home Gateway Unit，家庭网关单元）两种类型。ONU的一端通过光纤连接到分光器，另一端通过有线或无线方式连接家里的终端设备。
2. ODN（Optical Distribution Network，光分配网络）：通过光纤和分光器为ONU和OLT提供“传输通道”，分光器可以将一路光信号以一定的比例分成多路。
3. OLT（Optical Line Terminal，光线路终端）：离用户最远，是PON的核心设备，一般安装部署在运营商的机房中，用于将用户数据进行汇总并上送到城域网传输。