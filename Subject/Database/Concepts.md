<img src="../../Pic/Subject/Database/database-overview.png" style="width:400px;padding:10px;"/>

## 核心组件
1. 进程管理器（process manager）：很多数据库具备进程/线程池。为了实现纳秒级操作，一些现代数据库使用自己的线程而不是操作系统线程。
2. 网络管理器（network manager）：网路I/O是个大问题，尤其对于分布式数据库。所以一些数据库具备自己的网络管理器。
3. 文件系统管理器（File system manager）：磁盘I/O是数据库的首要瓶颈。需要文件系统管理器来处理OS文件系统甚至取代OS文件系统。
4. 内存管理器（memory manager）：为了避免磁盘I/O带来的性能损失，需要大量的内存。如果要处理大容量内存需要高效的内存管理器，尤其是同时有很多查询的时候。
5. 安全管理器（Security Manager）：用于对用户的验证和授权。
6. 客户端管理器（Client manager）：用于管理客户端连接。
## 工具
1. 备份管理器（Backup manager）：用于保存和恢复数据。
2. 复原管理器（Recovery manager）：用于崩溃后重启数据库到一个一致状态。
3. 监控管理器（Monitor manager）：用于记录数据库活动信息和提供监控数据库的工具。
4. Administration管理器（Administration manager）：用于保存元数据（比如表的名称和结构），提供管理数据库、模式、表空间的工具。
## 查询管理器
1. 查询解析器（Query parser）：用于检查查询是否合法
2. 查询重写器（Query rewriter）：用于预优化查询
3. 查询优化器（Query optimizer）：用于优化查询
4. 查询执行器（Query executor）：用于编译和执行查询
## 数据管理器
1. 事务管理器（Transaction manager）：用于处理事务
2. 缓存管理器（Cache manager）：数据被使用之前置于内存，或者数据写入磁盘之前置于内存
3. 数据访问管理器（Data access manager）：访问磁盘中的数据

## 数据库概念
1. 字段（列）：某一个事物的一个特征，如姓名。
2. 记录（元组）：事物特征的组合，可以描述一个具体的事物，即某一行。
3. 主键：能唯一标识信息的事物。