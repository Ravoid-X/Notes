## char a,char a[],char *a,char *[],char **a 
### char a
定义了一个存储空间，存储的是char类型的变量
### char a[]
字符数组，数组中的每一个元素是一个char类型的数据
### char *a
1. 字符串的本质（在计算机眼中）是其第一个字符的地址，所以和 char a[] 没有本质区别
2. 对于 char s[] 和 char* a，可以有a=s，但不能有s=a。
3. 创建数组的时候 s 的地址不为空已经确定，但是 a 是一个空指针，不能将非空的地址指向空指针。
### char *a[]
1. * 的优先级是低于 [] 的，因此要先看 a[] 再看 *
2. 一个 char 数组，数组中的每一个元素都是指针，这些指针指向 char 类型
3. `char *a[ ] = {"China","French","America","German"}`
### char **a
1. 两个 ** 代表相同的优先级，因此从右往左看，即 char*(*a)
2. 和 char *a[] 一样