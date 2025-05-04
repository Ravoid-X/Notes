## 定义（pending）
视图（View）是一种虚拟存在的表。视图中的数据并不在数据库中实际存在，行和列数据来自定义视图的查询中使用的表，并且是在使用视图时动态生成的。通俗的讲，视图只保存了查询的SQL逻辑，不保存查询结果。
## 语法
1. 创建
```
CREATE [OR REPLACE] VIEW 视图名称[(列名列表)] AS SELECT语句 [ WITH [CASCADED | LOCAL ] CHECK OPTION ]
```
2. 查询
```
SHOW CREATE VIEW 视图名称; //查看创建视图语句
SELECT * FROM 视图名称 ...... ; //查看视图数据
```
3. 修改
```
CREATE [OR REPLACE] VIEW 视图名称[(列名列表)] AS SELECT语句 [ WITH [ CASCADED | LOCAL ] CHECK OPTION ]
```
```
ALTER VIEW 视图名称[(列名列表)] AS SELECT语句 [ WITH [ CASCADED | LOCAL ] CHECK OPTION ]
```
4. 删除
```
DROP VIEW [IF EXISTS] 视图名称 [,视图名称] ...
```
5. 示例
```
-- 创建视图
create or replace view stu_v_1 as select id,name from student where id <= 10;
-- 查询视图
show create view stu_v_1;
select * from stu_v_1;
select * from stu_v_1 where id < 3;
-- 修改视图
create or replace view stu_v_1 as select id,name,no from student where id <= 10; 
alter view stu_v_1 as select id,name from student where id <= 10;
-- 删除视图
drop view if exists stu_v_1;
```
6. 插入
```
insert into stu_v_1 values(6,'Tom');
```