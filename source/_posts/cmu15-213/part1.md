---
title: CMU 15-213 Part1 Bit & Int
date: "2020/1/17"
categories:
- CMU 15-213
---

### 二进制表示法

* 用bits实现set数据结构

* 位运算(&, |, ~, ^)与逻辑运算的区分(!, &&, ||)

!!0x55 => !(0x00) => 0x01

### 有/无符号类型

* 有符号int的范围: [0, 2<sup>32</sup>-1], unsigned int: [-2<sup>31</sup>, 2<sup>31</sup>-1]

* 2的补码

![补码](https://i.loli.net/2020/01/12/Zyp3oN2gvunIFas.png)

* 赋值中的显式/隐式类型转换: 二进制位不会发生改变，只改变解释方式

* 表达式中的类型转换：只要存在unsigned，signed会自动转换成unsigned

```
int x, y;
unsigned ux, uy;

ux = (unsigned)x;   // explicit casting 
y = uy;             // implicit casting
```

试考虑下列比较关系：

1. **2147483647U** < **-2147483648** => true  

发生自动类型转换，-2147483648转换成unsigned类型，**二进制不发生改变，只改变解释方式**，所以-2147483648变成2147483648U；

2. for (unsigned i=n-1;i>=0;i--) {...} => 死循环

i 作为unsigned类型，循环条件>=0永远成立；

3. `for (int i=n-1;i-sizeof(char)>=0;i--) {...}` =>死循环

表达式i-sizeof(char)中sizeof返回的size_t是unsigned类型，i自动转型成unsigned类型，表达式返回unsigned类型，循环条件永远不会停止；

4 #define ABS(x) ((x)>0?(x):-(x)) => 在x=Tmin时溢出

比如输入-2147483648, 仍得到-2147483648

### 运算

* 位的扩展/截取

扩展unsigned总是在最高位补充0，扩展signed总是在最高位补充符号位的值。

![unsigned截取](https://i.loli.net/2020/01/12/JQlrqwAaUOyYN7H.png)

![signed的扩展](https://i.loli.net/2020/01/12/IH4DtomU3ncNJpj.png)

截取unsigned相当于mod运算，截取正的signed相当于mod，负的就...

![正signed截取](https://i.loli.net/2020/01/12/qGdwRZABolxjngE.png)

![负signed截取](https://i.loli.net/2020/01/12/qeuhSQZHNEKjF29.png)


* 加法

如果变量可表示的范围无法容纳运算结果，那么会发生溢出。值得注意的是，虽然大多数signed溢出的行为为`INT_MAX+1 => INT_MIN`，但是C语言标准并没有规定signed类型发生溢出的行为，依赖这个特性的实现不能得到保证。然而，C语言标准规定了unsigned类型发生溢出的行为，即`UINT_MAX+1=0`，我们可以依赖这个特性实现类似取模的效果：

```
for (size_t i=cnt-1;i<cnt;i--) {...}
```

注意，当`cnt<0`时，该循环就无法按照意图运行。

![无符号加法溢出](https://i.loli.net/2020/01/16/G1Wk2LXxpsdM3ol.png)

![有符号加法溢出](https://i.loli.net/2020/01/16/AfjeFTUidRpB46a.png)

* 乘法

在汇编中，signed和unsigned使用不同的指令进行乘法运算，并且两个w位的数相乘得到一个2w的数。但在C语言中，signed和unsigned使用相同的乘法运算，两个w位的数相乘会舍弃高w位的部分，只保留低w位。无论是无符号正数，还是有符号补码负数，都采用同一种方式进行乘法运算，而且可以得到相同的结果，无需对符号位的额外处理。而舍弃高位的做法使结果更有可能溢出，即正数乘正数得到一个负数，或者反之。

* 位移

存在逻辑位移和算术位移两种位移运算，逻辑位移在新进入的位补0，算术位移在新进入的位补符号位。大多数情况中，移位与对2<sup>k</sup>的乘除法等价，编译器会把后者优化为前者，因为移位运算只需要1个时钟周期，而乘除法运算需要至少3个时钟周期。这里的大多数情况，其实指的是无符号数与有符号正数的乘除法。对于有符号负数补码的乘法，只有在C语言实现的左移（<<运算）为**算术左移**，等价关系才成立。幸运的是，大多数C语言实现都是算术左移。而对于有符号的负数补码的除法，等价关系并不能成立。

![负数补码左移与除法](https://i.loli.net/2020/01/16/8eSN1rgjsLn749v.png)

当移位长度小于0或者大于等于8是未定义的行为，如在大多数机器实现中, 0x07 >> 8 => 0x07 (≠> 0)，原因是某些机器实现移位长度只取末3位。


### 字节在内存中的表示

面向程序员的内存模型是一个背靠背排列的大型数组。在字长64位机器上，程序员通过一个64位大小的指针（实际只用47位可用）来读写内存上的数据。字长是计算机系统的一个参数，对于程序员来说，字长大小即是指针大小；而对于系统底层来说，字长是CPU处理/总线传输数据的单位大小。根据字长，内存被分割成字，而最低字节的地址作为字的地址。

![类型大小](https://i.loli.net/2020/01/16/czIw2WGNofVJXUE.png)

* 大端与小端模式

一般我们使用的机器和操作系统（x86/x86-64上的MacOS/Windows/Linux，ARM上的Android/iOS）都是小端模式，即LSB放在低地址。而网络传输往往采用大端模式，即LSB放在高地址。所以中间过程要进行转换。

* 字符串在内存中

![字符串](https://i.loli.net/2020/01/16/Smd1HbUO9gr6GIi.png)

