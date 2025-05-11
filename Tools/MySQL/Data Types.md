## 数值
<img src="../../Pic/Tools/MySQL/mysql-data-numeric.png" style="width:900px;padding:10px;"/>

1. INT(11) 中的数字只是规定了交互工具显示字符的个数，对于存储和计算来说是没有意义的。
2. DECIMAL 为高精度小数类型。CPU 原生不支持 DECIMAl 类型的计算，计算 DECIMAl 比浮点类型需要更高的代价。
3. FLOAT、DOUBLE 和 DECIMAL 都可以指定列宽，例如 DECIMAL(18, 9) 表示总共 18 位，取 9 位存储小数部分，剩下 9 位存储整数部分。

## 日期和时间
<img src="../../Pic/Tools/MySQL/mysql-data-time.png" style="width:900px;padding:10px;"/>

1. DATETIME：能够保存从 1001 年到 9999 年的日期和时间，精度为秒，使用 8 字节的存储空间，与时区无关。\
默认情况下，MySQL 以一种可排序的、无歧义的格式显示 DATETIME 值，例如“2008-01-16 22:37:08”
2. TIMESTAMP：保存从 1970 年 1 月 1 日午夜（格林威治时间）以来的秒数，使用 4 个字节，只能表示从 1970 年 到 2038 年，和时区有关。\
MySQL 提供了 FROM_UNIXTIME() 函数把 UNIX 时间戳转换为日期，并提供了 UNIX_TIMESTAMP() 函数把日期转换为 UNIX 时间戳。\
默认情况下，如果插入时没有指定 TIMESTAMP 列的值，会将这个值设置为当前时间。
3. 尽量使用 TIMESTAMP，因为它比 DATETIME 空间效率更高。

## 字符串
<img src="../../Pic/Tools/MySQL/mysql-data-string.png" style="width:900px;padding:10px;"/>

1. VARCHAR 这种变长类型能够节省空间，因为只需要存储必要的内容。\
但是在执行 UPDATE 时可能会使行变得比原来长，当超出一个页所能容纳的大小时，就要执行额外的操作。\
MyISAM 会将行拆成不同的片段存储，而 InnoDB 则需要分裂页来使行放进页内。
2. VARCHAR 会保留字符串末尾的空格，而 CHAR 会删除。

## 选择优化的数据类型
1. 选择更小的数据类型
2. 整形比字符串操作代价更低；使用内建类型而不是字符串来存储日期和时间；用整形存储IP地址等。
3. 尽量避免NULL：如果查询中包含可为NULL的列，对MySQL来说更难优化，因为可为 NULL 的列使得索引、索引统计和值比较都更复杂。

## 选择标识符（identifier）
1. 整数类型通常是标识列的最佳选择，因为它们很快并且可以使用AUTO_INCREMENT。
2. 尽量避免使用字符串类型作为标识列，因为它们很耗空间，并且比数字类型慢。
3. 如MD5(),SHA1()或者UUID()产生的完全随机字符串，会任意分布在很大的空间内，这会导致INSERT以及一些SELECT语句变得很慢。
4. 因为插入值会随机的写入到索引的不同位置，所以使得INSERT语句更慢。这会导致叶分裂、磁盘随机访问。
5. SELECT语句会变的更慢，因为逻辑上相邻的行会分布在磁盘和内存的不同地方。
6. 随机值导致缓存对所有类型的查询语句效果都很差，因为会使得缓存赖以工作的局部性原理失效。