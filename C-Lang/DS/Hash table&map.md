## 定义
1. 散列表是一种不比较key，而是根据key计算key在表中的位置的数据结构；是key和其所在存储地址的映射关系，散列表通过此方式达到快速索引的目的。
2. 由hash函数和数组构成，以<key, value>存储\
（1）hash函数：映射，通过key找到其存储地址。\
（2）数组：key通过hash函数找到数组存储value的位置，叫做散列表。
3. 常用结构
<img src="../../Pic/C-Lang/DS/hash-structure.png" style="width:400px;padding:10px;"/>

## hash
1. 遵循原则\
（1）计算速度快\
（2）强随机分布（等概率、均匀的分布在整个地址空间）
2. 满足的函数\
(1)murmurhash1：计算速度非常快，但质量一般。\
(2)murmurhash2：计算速度较快，质量也可以。\
(3)murmurhash3：计算速度较慢，但质量很好。\
(4)siphash：redis 6.0中使用，因为redis设置的key非常有规律而且字符串接近，siphash 主要解决字符串接近的强随机分布性。
3. hash冲突：可能会把两个或两个以上的不同key映射到同一地址。常用的解决方法有开放地址法和链表法。
4. 冲突原因\
（1）把n+1个元素放入n大小的数组\
（2）hash是随机的，产生的数对数组长度取余很可能相同
5. 负载因子=数组存储元素的个数 / 数组长度，用于描述冲突的激烈程度和存储的密度。
6. 对于小型的哈希表，双哈希法要比二次探测的效果好。如果内存充足并且哈希表一经创建，就不再修改其容量，线性探测效果相对比较好。
7. 创建哈希表时，不知道未来存储的数据有多少，使用链表法要比开放地址法好，如果使用开放地址法，随着装载因子的变大，性能会直线下降。

## 开放地址法
若数据不能直接存放在哈希函数计算出来的数组下标时，就需要寻找其他位置来存放。有三种方式：线性探测、二次探测和再哈希法。
1. 线性探测\
线性的查找空白单元，如得到的数组下标已有对象，则一直往下找直到一个空白地方。\
（1）插入
```
void insert(Student student){
    int key = student.getKey();
    int hashVal = hash(key);
    while (array[hashVal] !=null && array[hashVal].getKey() !=-1){
        ++hashVal;
        // 如果超过数组大小，则从第一个开始找
        hashVal %=size;
    }
    array[hashVal] = student;
}
```
（2）查找
```
Student find(int key){
    int hashVal = hash(key);
    while (array[hashVal] !=null){
        if (array[hashVal].getKey() == key){
            return array[hashVal];
        }
        ++hashVal;
        hashVal %=size;
    }

    return null;
}
```
（3）删除\
通过线性探测方法找到一个空闲位置，我们就可以认定哈希表中不存在这个数据。如果这个空闲位置是后来删除的，就会导致原来的查找算法失效。需要一个特殊的数据来顶替这个被删除的数据。\
如何在线性探测哈希表中做了多次操作，会导致哈希表中充满了学号为-1的数据项，使的哈希表的效率下降，所以很多哈希表中没有提供删除操作。
```
Student delete(int key){
    int hashVal = hash(key);
    while (array[hashVal] !=null){
        if (array[hashVal].getKey() == key){
            Student temp = array[hashVal];
            array[hashVal]= noStudent;
            return temp;
        }
        ++hashVal;
        hashVal %=size;
    }
    return null;
}
```
2. 二次探测\
（1）在线性探测哈希表中，数据会发生聚集，一旦聚集形成，它就会变的越来越大。二次探测中，探测过程是x+1,x+4,x+9,x+16......\
（2）二次探测也产生了新的聚集问题，所有映射到同一位置的关键字在寻找空位时，探测的位置都是一样的。比如讲1、11、21依次插入到哈希表中，它们映射的位置都是1，那么11需要以一为步长探测，21需要以四为步长探测，31需要为九为步长探测，成为二次聚集。
3. 再哈希法\
增加一个哈希函数用来根据关键字生成探测步长，需和第一个哈希函数不一样且不能输出0。stepSize = constant-(key%constant)函数效果非常好，constant是一个质数并且小于数组容量。
（1）选择质数是因为探测过程会查找到哈希表每一个位置，如果容量为15，步长为5，若没找到会循环查找0，5，10
（2）插入
```
int stepHash(int key) {
    return 7 - (key % 7);
}
void insert(Student student) {
    int key = student.getKey();
    int hashVal = hash(key);
    // 获取步长
    int stepSize = stepHash(key);
    while (array[hashVal] != null && array[hashVal].getKey() != -1) {
        hashVal +=stepSize;
        // 如果超过数组大小，则从第一个开始找
        hashVal %= size;
    }
    array[hashVal] = student;
}
```
（3）查找
```
Student find(int key) {
    int hashVal = hash(key);
    int stepSize = stepHash(key);
    while (array[hashVal] != null) {
        if (array[hashVal].getKey() == key) {
            return array[hashVal];
        }
        hashVal +=stepSize;
        hashVal %= size;
    }
    return null;
}
```
（4）删除
```
Student delete(int key) {
    int hashVal = hash(key);
    int stepSize = stepHash(key);
    while (array[hashVal] != null) {
        if (array[hashVal].getKey() == key) {
            Student temp = array[hashVal];
            array[hashVal] = noStudent;
            return temp;
        }
        hashVal +=stepSize;
        hashVal %= size;
    }
    return null;
}
```
## 链表法
（1）每个数组对应一条链表。当value落到哈希表中的某个位置，将其添加到链表，其他同样映射到这个位置的数据项也只需添加到链表。\
（2）可以使用有序链表，不能加快成功的查找，但是可以减少不成功的查找时间。\
（3）插入
```
void insert(Link link){
        int key = link.getKey();
        Link previous = null;
        Link current = first;
        while (current!=null && key >current.getKey()){
            previous = current;
            current = current.next;
        }
        if (previous == null)
            first = link;
        else
            previous.next = link;
        link.next = current;
    }
```
（4）查找
```
Link find(int key){
        Link current = first;
        while (current !=null && current.getKey() <=key){
            if (current.getKey() == key){
                return current;
            }
            current = current.next;
        }
        return null;
    }
```
（5）删除
```
void delete(int key){
        Link previous = null;
        Link current = first;
        while (current !=null && key !=current.getKey()){
            previous = current;
            current = current.next;
        }
        if (previous == null)
            first = first.next;
        else
            previous.next = current.next;
    }
```