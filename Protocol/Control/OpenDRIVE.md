## 概述（pending）
1. ASAM OpenDRIVE 是一种用于高精度、可解析描述道路网络的开放式、标准化 XML 文件格式
2. 核心目的是为仿真提供静态道路环境的“数字地基”，不描述动态内容。
3. 采用参数化而非网格描述，可以无限放大而不失真，道路、车道、曲率等所有信息都可以通过数学公式直接查询，而不是通过采样点反算。
## 坐标系
使用一种基于参考线的相对坐标系，地图中的每一条“路” ($<road>$) 都是围绕一条中心参考线来构建的
1. 首先定义一条路的几何路径（直线、弧线、螺旋线等）
2. 在这条参考线的左侧和右侧“附加”车道
3. 最后将交通标志、路灯等对象放置在这条路的坐标系中
## 结构
### `<header>`
头部信息，包含版本、日期、地理参考（如坐标系）等元数据
### `<road>`
定义了一段连续的道路，是 OpenDRIVE 中最核心的元素
### `<junction>`
交叉口的定义，用于连接多条路
## `<road>` 元素：道路的构建块
### 组成
1. `<planView>`：定义道路在 $X/Y$ 平面上的几何形状（即参考线）。
2. `<lanes>`：定义附着在参考线上的车道布局。
3. `<signals> / <objects>`：定义路上的交通信号（如红绿灯）和固定对象（如路灯、障碍物）
### 示例 1 
一条 100 米长、双向两车道的直线公路
```<OpenDRIVE>
    <header ... />
    <road name="Simple Straight Road" length="100.0" id="1" junction="-1">
        <planView>
            <geometry s="0.0" x="0.0" y="0.0" hdg="0.0" length="100.0">
                <line/>
            </geometry>
        </planView>
        <lanes>
            <laneSection s="0.0">
                <center>
                    <lane id="0" type="none" level="false">
                        <roadMark type="solid" width="0.15" />
                    </lane>
                </center>
                <right>
                    <lane id="-1" type="driving" level="false">
                        <width sOffset="0.0" a="3.5" b="0" c="0" d="0" />
                        <roadMark type="solid" width="0.15" />
                    </lane>
                </right>
                <left>
                    <lane id="1" type="driving" level="false">
                        <width sOffset="0.0" a="3.5" b="0" c="0" d="0" />
                        <roadMark type="broken" width="0.15" />
                    </lane>
                </left>
            </laneSection>
        </lanes>
    </road>
</OpenDRIVE>
```
1. length="100.0" 定义了总长度。id="1" 是这条路的唯一标识。junction="-1" 是一个特殊值，表示这条路不属于任何交叉口。
2. 
## `<junction>` 元素：连接道路
### 



