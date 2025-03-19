## 中文编码
Python2.X默认编码格式是 ASCII，无法正确打印汉字\
在开头加入`# coding=utf-8`或者`# -*- coding: UTF-8 -*-`
## 缩进
使用缩进来指示代码块，必须在同一代码块中使用相同数量的空格
## 注释
\#单行注释\
"""多行注释，Python 会忽略未分配给变量的字符串文字
## 变量
1. 没有声明变量的命令，首次赋值时才会创建变量
2. 不需要声明类型，甚至可在设置后更改其类型
3. 字符串变量可以使用单引号或双引号进行声明
4. 多变量赋值
    ````
    x, y, z = "Orange", "Banana", "Cherry"
    x = y = z = "Orange"
    ````
5. 输出变量\
组合数字和字符串会报错
    ````
    x = "awesome"
    print("Python is " + x)
    ````
6. 变量作用域\
函数内部创建具有相同名称的变量，则该变量将是局部变量\
只能在函数内部使用。具有相同名称的全局变量将保留原样，并拥有原始值\
函数内部创建全局变量，使用 global 关键字\
函数内部更改全局变量，使用 global 关键字引用该变量
# 数据类型
&emsp;&emsp;文本类型：	str\
&emsp;&emsp;数值类型：	int, float, complex\
&emsp;&emsp;序列类型：	list, tuple, range\
&emsp;&emsp;映射类型：	dict\
&emsp;&emsp;集合类型：	set, frozenset\
&emsp;&emsp;布尔类型：	bool\
&emsp;&emsp;二进制类型：	bytes, bytearray, memoryview
## 字符串
所有字符串方法都返回新值。它们不会更改原始字符串
1. 多行字符串\
三个双引号或三个单引号
    ````
    a = """Python is a widely used general-purpose, high level programming language. 
    and its syntax allows programmers to express concepts in fewer lines of code."""
    print(a)
    ````
2. 裁切\
获取从位置 2 到位置 5（不包括）的字符
    ````
    b = "Hello, World!"
    print(b[2:5]) #llo
    ````
3. 负的索引\
-1 表示最后一个，-2 表示倒数第二个
    ````
    b = "Hello, World!"
    print(b[-5:-2]) #orl
    ````
4. strip() 删除开头和结尾的空白字符
5. lower() 返回小写的字符串;upper() 方法返回大写
6. replace() 用另一段字符串来替换字符串
    ````
    a = "Hello, World!"
    print(a.replace("World", "Kitty"))
    ````
7. split() 方法在找到分隔符的实例时将字符串拆分为子字符串
    ````
    a = "Hello, World!"
    print(a.split(",")) # returns ['Hello', ' World!']
    ````
8.  in 或 not in 关键字检查字符串中是否存在特定短语或字符
    ````
    txt = "China is a great country"
    x = "ain" not in txt
    ````
9. format() 组合字符串和数字
    ````
    quantity = 3
    itemno = 567
    myorder = "I want {} pieces of item {}"
    print(myorder.format(quantity, itemno))
    ````
## bool
&emsp;bool() 可评估任何值，并返回 True 或 False\
1. 除空字符串外，任何字符串均为 True
2. 除 0 外，任何数字均为 True
3. 除空列表外，任何列表、元组、集合和字典均为 True
4. 空值（例如 ()、[]、{}、""、数字 0 和值 None）为False
## 列表（List）
&emsp;有序且可更改的集合，用方括号编写
1. 创建
    ````
    thislist = ["apple", "banana", "cherry"]
    thislist = list(("apple", "banana", "cherry"))
    ````
2. 更改
    ````
    thislist[1] = "mango"
    ````
3. 添加\
   append()将项目添加到列表的末尾; insert()在指定索引处添加项目
    ````
    thislist.append("orange")
    thislist.insert(1, "orange")
    ````
4. 删除\
    remove() 删除指定项目;pop() 删除指定索引，如未指定则删除最后一项，返回值是被删除的项目
    ````
    thislist.remove("banana")
    thislist.pop()
    ````
    del 关键字删除指定的索引或完整删除列表;clear() 清空列表
    ````
    del thislist[0]
    del thislist        #列表删除
    thislist.clear()    #列表还存在，元素为空
    ````
5. 复制\
   list2 = list1,list2 只是对 list1 的引用，list1 的更改影响 list2\
   可使用 copy() 和 list() 制作副本
    ````
    mylist1 = thislist.copy()
    mylist2 = list(thislist)
    ````
6. 合并
    ````
    list1 = ["a", "b" , "c"]
    list2 = [1, 2, 3]
    list3 = list1 + list2
    for x in list2:
        list1.append(x)
    list1.extend(list2)
    ````
## 元组（Tuple）
&emsp;有序且不可更改的集合,用圆括号编写
1. 创建
    ````
    thistuple = ("apple", "banana", "cherry")
    thistuple = tuple(("apple", "banana", "cherry"))
    thistuple = ("apple",) #创建一个项目的元组，必须添加逗号，否则无法识别为元组
    ````
2. 更改\
    元组是不可变的，但可将元组转换为列表，更改列表，再转换回元组
    ````
    x = ("apple", "banana", "cherry")
    y = list(x)
    y[1] = "kiwi"
    x = tuple(y)
    ````
3. 删除\
    无法删除元组中的项目,但可完全删除元组
    ````
    del thistuple
    ````
4. 合并
    ````
    tuple3 = tuple1 + tuple2
    ````
## 集合（Set）
&emsp;无序和无索引的集合，用花括号编写
1. 创建
    ````
    thisset = {"apple", "banana", "cherry"}
    thisset = set(("apple", "banana", "cherry"))
    print(thisset) #无法确定项目的显示顺序
    ````
2. 访问\
    无法通过索引访问，但可使用 for 遍历集合，或使用 in 查询是否存在某值
3. 添加\
    集合一旦创建，无法更改项目，但可添加新项目\
    add()添加单项，update()添加多项
    ````
    thisset.add("orange")
    thisset.update(["orange", "mango", "grapes"])
    ````
4. 删除\
    remove()如项目不存在将引发错误，discard()如项目不存在不会引发错误\
    还可使用 pop() 删除随机项，clear() 方法清空集合，del 彻底删除集合
    ````
    thisset.remove("banana")
    thisset.discard("banana")
    ````
5. 合并\
    union() 返回一个新集合，包含两个集合中所有项目\
    update() 将 set2 中的项目插入 set1 中\
    union() 和 update() 都将排除任何重复项
    ````
    set3 = set1.union(set2)
    set1.update(set2)
    ````

## 词典（Dictionary）
&emsp;无序、可变和有索引的集合,用花括号编写，拥有键和值
1. 创建
    ````
    thisdict =	{
        "brand": "Porsche",
        "year": 1963
    }
    thisdict = dict(brand="Porsche", year=1963)
    ````
2. 访问
    ````
    x = thisdict["year"]
    x = thisdict.get("year")
    ````
3. 更改/添加
    ````
    thisdict["year"] = 2019
    ````
4. 遍历\
    循环遍历字典时，返回值是字典的键
    ````
    for x in thisdict:
        print(x)            #打印键名
        print(thisdict[x])  #打印值
    for x in thisdict.values():     #返回值
    for x, y in thisdict.items():   #返回键和值
    ````
5. 删除\
    del 和 pop() 删除指定键名的项\
    popitem() 删除最后插入项目（在 3.7 之前版本，删除随机项目）
    ````
    thisdict.pop("year")
    del thisdict["year"]
    thisdict.popitem()
    ````
6. 复制\
    dict2 = dict1,dict2 只是对 dict1 的引用，dict1 的更改影响 dict2\
    ````
    mydict = thisdict.copy()
    mydict = dict(thisdict)
    ````
7. 嵌套
    ````
    myfamily = {
        "child1" : {
            "name" : "Phoebe Adele",
            "year" : 2002
        },
        "child2" : {
            "name" : "Rory John",
            "year" : 1999
        }
    }
    ````
    ````
    child1 = {
        "name" : "Phoebe Adele",
        "year" : 2002
    }
    child2 = {
        "name" : "Jennifer Katharine",
        "year" : 1996
    }
    myfamily = {
        "child1" : child1,
        "child2" : child2
    }
    ````
# 函数
## if Else
1. 多行
    ````
    if b > a:
        print("b is greater than a")
    elif a == b:
        print("a and b are equal")
    else:
        print("a is greater than b")
    ````
2. 单行
    ````
    if a > b: print("A")
    print("A") if a > b else print("B")
    print("A") if a > b else print("=") if a == b else print("B")
    ````
3. pass\
    if 不能为空，但如写了无内容的 if 语句，使用 pass 避免错误
    ````
    if b > a:
        pass
    ````
## for
&emsp;用于迭代序列（即列表，元组，字典，集合或字符串）
1. 循环指定次数
    ````
    for x in range(10):     #值 0 到 9
    for x in range(3, 10):  #值 3 到 9
    for x in range(2, 30, 3):  #步长3
        print(x)
    ````
2. 循环结束\
    else 关键字指定循环结束时要执行的代码块
    ````
    for x in range(10):
        print(x)
    else:
        print("Finally finished!")
    ````
## 函数
1. 定义/调用
    ````
    def my_function():
        print("Hello from a function")

    my_function()
    ````
2. 传参
    ````
    def my_function(fname):
        print(fname + " Gates")

    my_function("Bill")
    ````
3. 默认参数
    ````
    def my_function(country = "China"):
        print("I am from " + country)

    my_function("Sweden")
    my_function()
    ````
4. List 传参\
    参数可以是任何数据类型，且在函数内视为相同数据类型
    ````
    def my_function(food):
    for x in food:
        print(x)

    fruits = ["apple", "banana", "cherry"]
    my_function(fruits)
    ````
5. 关键字传参
    ````
    def my_function(child3, child2):
        rint("The youngest child is " + child1)

    my_function(child1 = "Phoebe", child2 = "Jennifer")
    ````
6. 任意参数
    ````
    def my_function(*kids):
        print("The youngest child is " + kids[1])

    my_function("Phoebe", "Jennifer")
    ````
7. Lambda\
    是一种小的匿名函数, 可接受任意数量的参数，但只有一个表达式
    ````
    x = lambda a, b : a * b
    print(x(5, 6))
    ````
    同一程序中使用相同的函数定义来生成两个函数
    ````
    def myfunc(n):
        return lambda a : a * n

    mydoubler = myfunc(2)
    mytripler = myfunc(3)

    print(mydoubler(11)) 
    print(mytripler(11))
    ````
## 递归
````
def tri_recursion(k):
  if(k>0):
    result = k+tri_recursion(k-1)
    print(result)
  else:
    result = 0
  return result

tri_recursion(6)
````
## 类&对象
&emsp;Python 中几乎所有东西都是对象，拥有属性和方法
1. \_\_init__()\
    所有类都有 \_\_init__()，创建新对象时会自动调用
    ````
    class Person:
        def __init__(self, name, age):
            self.name = name
            self.age = age
        def myfunc(abc):
            print("Hello my name is " + abc.name)
    p1 = Person("Bill", 63)
    ````
2. self
    是对类的当前实例的引用\
    不必被命名为 self，但必须是类中任意函数的首个参数
## 继承
1. 创建子类
    ````
    class Student(Person):
        pass
    ````
2. \_\_init__()\
    子 \_\_init__() 会覆盖对父 \_\_init__() 的继承\
    需保持父 \_\_init__() 的继承，添加对父的 \_\_init__() 的调用
    ````
    class Student(Person):
        def __init__(self, fname, lname):
            Person.__init__(self, fname, lname)
    ````
    super() 会使子类从其父继承所有方法和属性
    ````
    class Student(Person):
        def __init__(self, fname, lname):
            super().__init__(fname, lname)
    ````
3. 添加属性\
    在子类中添加一个与父类中的函数同名的方法，则将覆盖父方法的继承
    ````
    class Student(Person):
        def __init__(self, fname, lname, year):
            super().__init__(fname, lname)
            self.graduationyear = year

    x = Student("Elon", "Musk", 2019)
    ````
## 迭代器
&emsp;是一种对象，包含值的可计数数字
1. 创建
    ````
    mytuple = ("apple", "banana", "cherry")
    myit = iter(mytuple)
    print(next(myit))
    ````
    \_\_iter__()可以执行操作（初始化等），必须返回迭代器对象本身\
    \_\_next__()允许执行操作，必须返回序列中的下一个项目
    ````
    class MyNumbers:
        def __iter__(self):
            self.a = 1
            return self
        def __next__(self):
            x = self.a
            self.a += 1
            return x

    myclass = MyNumbers()
    myiter = iter(myclass)
    print(next(myiter))
    ````
2. for\
    for 循环实际上创建了一个迭代器对象，并为每个循环执行 next() 方法
3. StopIteration\
    防止迭代永远进行
    ````
    def __next__(self):
        if self.a <= 20:
            x = self.a
            self.a += 1
            return x
        else:
            raise StopIteration
    ````
## 字符串格式化
有时文本的一部分来自数据库或用户输入\
要控制此类值，添加占位符（花括号 {}），再通过 format() 方法运行值
1. 指定类型\
    在花括号内添加参数以指定如何转换值
    ````
    txt = "The price is {:.2f} dollars"
    print(txt.format(price))
    ````
2. 多值
    ````
    myorder = "I want {} pieces of item number {} for {:.2f} dollars."
    print(myorder.format(quantity, itemno, price))
    ````
    或添加索引
    ````
    txt = "His name is {1}. {1} is {0} years old."
    print(txt.format(age, name))
    ````S
    命名索引
    ````
    myorder = "I have a {carname}, it is a {model}."
    print(myorder.format(carname = "Porsche", model = "911"))
    ````