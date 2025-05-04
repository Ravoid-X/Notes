## 第一范式
列的原子性，即列不能够再分成其他几列。

## 第二范式
满足 1NF，另外包含两部分内容，一是表必须有一个主键；二是非主键字段必须完全依赖于主键，而不能只依赖于主键的一部分。
<img src="../../../Pic/Tools/Database/MySQL/mysql-2th-nf.png" style="width:500px;padding:10px;"/>

1. Discount（折扣），Quantity（数量）完全依赖于主键（OrderID），而 UnitPrice单价，ProductName产品名称 只依赖于 ProductID, 所以 OrderDetail 表不符合 2NF。
2. 可以把【OrderDetail】表拆分为【OrderDetail】（OrderID，ProductID，Discount，Quantity）和【Product】（ProductID，UnitPrice，ProductName）这样就符合第二范式了。

## 第三范式
满足 2NF，另外非主键列必须直接依赖于主键，不能存在传递依赖。即不能存在：非主键列 A 依赖于非主键列 B，非主键列 B 依赖于主键的情况。
<img src="../../../Pic/Tools/Database/MySQL/mysql-3th-nf.png" style="width:800px;padding:10px;"/>

1. OrderDate，CustomerID，CustomerName，CustomerAddr，CustomerCity 等非主键列都完全依赖于主键（OrderID），所以符合 2NF。不过问题是 CustomerName，CustomerAddr，CustomerCity 直接依赖的是 CustomerID（非主键列），而不是直接依赖于主键，它是通过传递才依赖于主键，所以不符合 3NF。
2. 可以把【Order】表拆分为【Order】（OrderID，OrderDate，CustomerID）和【Customer】（CustomerID，CustomerName，CustomerAddr，CustomerCity）从而达到 3NF。

## E-R模型
即实体-关系模型，E-R模型就是描述数据库存储数据的结构模型。效果图如下
<img src="../../../Pic/Tools/Database/MySQL/mysql-e-r.png" style="width:800px;padding:10px;"/>

1. 说明\
（1）实体: 用矩形表示，并标注实体名称\
（2）属性: 用椭圆表示，并标注属性名称\
（3）关系: 用菱形表示，并标注关系名称
2. 关系\
（1）一对一：在表A或表B中创建一个字段，存储另一个表的主键值。\
（2）一对多：在多的一方表(学生表)中创建一个字段，存储班级表的主键值。\
（3）多对多：新建一张表C，这个表只有两个字段，一个用于存储A的主键值，一个用于存储B的主键值。