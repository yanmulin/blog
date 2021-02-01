---
title: CMU 15-213 Part2 Floating Points
date: "2020/1/27"
categories:
- CMU 15-213
---

### 浮点数的二进制表示

浮点数将二进制数位分成3个部分，分别为s符号位，exp指数部分和frac小数部分。而浮点数有两种不同的精度类型，32位的float和64位的double。两者的符号位都只占用一位，而double型浮点数的指数部分和小数部分比float型长，所以double型能表示更高精度的浮点数，相应的也占用了更多内存空间。

![float和double.png](https://i.loli.net/2020/01/27/H2tLvOroJwayIdn.png)

浮点数真实值可以用如下公式表示：

(-1)<sup>s</sup>⨉M⨉2<sup>E</sup>

其中s表示符号位，E是经过exp指数部分计算得到，M是经过frac小数部分计算得到。而浮点数存在两种模式：Normalized和Denormalized，前者表示距离0比较远的数，而后者表示距离0比较近的数。这两种模式中E和M的计算方式有所不同。

但exp不全为0且不全为1时为Normalized模式，在Normalized模式下：

M=1.frac(小数点左边隐含了1)
E=exp-Bias(Bias=2<sup>k-1</sup>-1, k=exp的位数)

而exp全为0时为Denormalized模式，在Denormalized模式下：

M=0.frac(小数点左边隐含了0)
E=1-Bias(Bias=2<sup>k-1</sup>-1, k=exp的位数)

这里采用Bias来计算E的原因是，使用Bias的表示方法可以把除符号位的浮点数当作unsigned直接比较大小。比如补码的最大值是0111，最小值是1111，占用浮点数中间数位时无法当作unsigned比较大小。而Denormalized模式下E的映射不采用与Normalized模式一致的0-Bias，而采用1-Bias的原因是，这样可以使得Denormalized和Normalized临界处数值的变化是连续的。通过把浮点数的所有值标记在数轴上的位置，可以看出这一点。

![数轴.png](https://i.loli.net/2020/01/27/ZrTuAwQ2vDKC57d.png)

同时，可以观察到Denormalized模式的浮点数是均匀地分布在0周围，而Normalized模式的浮点数不均匀地向±inf扩张。

### 浮点数的特殊值

浮点数有±0/±inf./NaN等5个特殊值，用来表示运算中比如溢出等的特殊情况。

* ±0: 与整型相同，exp和frac全为0时浮点数的真实值也为0。当时符号位可以取值0或1，这便导致浮点数存在两个不同的0，即±0。

* ±inf: 当exp全为1，frac全为0时，表示无穷大，根据符号位不同分为±inf.。与整型用取模来处理溢出不同，浮点数统一使用±inf.来表示溢出。另外除0，比如**3.0/0.0**, **-1.5/0.0**等表达式也会得到±inf.。

* NaN: 当exp全为1且frac不为0时表示非数字(Not A Number)，即NaN。如**sqrt(-1)**, **inf. + 1.0**, **inf. + -inf.**等非法表达式就会产生NaN的结果。

### 浮点数表示的范围

由上面数轴图可知，浮点数可以表示的值不是连续的，而是一个个离散的点。Denormalized模式表示靠近0的值，而Normalized模式表示远离0的值。而且Denormalized模式到Normalized模式的过渡是平滑的。

下面以一个5位（无符号位，3位exp指数部分，2位frac小数部分）浮点数为例计算该浮点数的范围。

(Bias=2<sup>3-1</sup>-1=3, Denormalized模式下E=1-Bias=-2)
* Denormalized最小值：000 01, 0.01<sub>2</sub>⨉2<sup>-2</sup>=1/4⨉1/4=**1/16**
* Denormalized最大值：000 11, 0.11<sub>2</sub>⨉2<sup>-2</sup>=3/4⨉1/4=**3/16**
* Normalized最小值：001 00, 1.00<sub>2</sub>⨉2<sup>1-3</sup>=(1+0)⨉1/4=**4/16**
* Normalized最大值：110 11, 1.11<sub>2</sub>⨉2<sup>6-3</sup>=(1+3/4)⨉8=**14**
* 浮点数最小值：**0**
* 浮点数最大值：**14**

### 浮点数的运算

由于浮点数能表示的数值是一个个离散的点，所以无法精确表示所有的计算结果。而考虑浮点数运算的思路是，**先进行精确计算，然后进行取整**。取整分为向下(-∞, round down)，向上(+∞, round up)，和向偶数(round to even)三种方式，默认采用向偶数取整的方式。向偶数取整基于统计学中数字均匀分布的假设，所以向偶数取整发生的概率为50%。向偶数取整只发生在尾数正好处于中间的位置(half way)，比如十进制下1.5向偶数取整位2.0，其余情况则作四舍五入考虑。

而在二进制下，偶数对应是最后有效位(guard bit)为0，尾数的中间位置(half way)对应只有尾数第一位为1其余为0，即100...

![取整示例.png](https://i.loli.net/2020/01/27/wX2igC51Rzph3ld.png)

对于加减乘除运算，先考虑精确的结果，然后用向偶数取整到可表示的精度范围。如果无法表示运算结果，则使用特殊值(±inf./NaN)。

