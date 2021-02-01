---
title: CMU 15-213 Part3 Machine Programming I
date: "2020/1/28"
categories:
- CMU 15-213
---


### 汇编基础

#### x86-64架构

计算机系统架构(Architecture)，指的是计算机的硬件系统对软件层（汇编语言开发者）的抽象，包括了处理器的寄存器设计和指令集。x86架构可以追溯到上世纪80年代Intel公司开发的80x86系列处理器，采用的是复杂指令集(CISC，相对于ARM芯片采用的精简指令集，RISC)。而x86-64架构是x86架构的64位版本，目前已经完全取代了32位的机器。从硬件角度来看，64位字长指的是CPU在一个时钟周期可以处理长度为64位的数据。x86-64架构提供了16个数据寄存器，除了个别有特殊用途（比如%rsp，用作栈顶指针），其余都可用来存放临时数据。除了数据寄存器，还有IP寄存器，指向下条指令的地址；以及状态码（Status Code)，有SF(Sign Flag)，ZF(Zero Flag)，CF(Carry Flag)等等，在代数计算指令执行完成后被自动设置表示结果的状态。

![x86-64架构寄存器一览](https://i.loli.net/2020/02/02/bc4fgr6WeEOPw5i.png)

#### 汇编的数据类型和取值方式

汇编语言中只用整型和浮点两种数据类型，整型分为1,2,4,8等四种字节长度，浮点分为单精度(float)和双精度(double)两种类型。x86-64架构下，汇编语言可以通过3种方式来读写数据：字面量，寄存器和内存。

* 字面量：相当于硬编码在程序中，如$1等

* 寄存器：可以通过寄存器的名字来访问，如%rax, %r12等

* 内存：格式为D(Rb, Ri, S)，D，Rb，Ri，S分别表示偏移(Displacement)，基址(Base)，下标(Index)，长度(Scale)，前三者可以取值为寄存器或字面量，而长度只可以取值字面量1,2,4,8。地址的计算公式为D+Rb+Ri*Scale。用法如4(%rbx, %rdi, 4), (%rbx), (,%rax,2)

#### 常用汇编指令

下图列出了x86-64中常用指令。

![x86-64常用指令](https://i.loli.net/2020/02/02/vr5RFedwBnDcaNU.png)

![x86-64常用指令](https://i.loli.net/2020/02/02/yYoQSqbAGMPIC4N.png)

* 计算数前者是源，后者是目标

* `movq src, dst`: 移动数据，注意不允许将字面量作为dst，或者src和dst都是内存取值。

* `leaq src, dst`: 本作为地址计算，但gcc常常用来优化代数计算，如`leaq (%rdi,%rdi,2), %rax`用来计算`t = x+2*x`

* `sarq`是算术右移(Arithmetic Shift)，`shrq`是逻辑右移(Logical Shift)


#### 浮点数

x86-64处理浮点数使用一套与整数不同的硬件和指令。

![浮点数专用寄存器](https://i.loli.net/2020/02/02/AUQnXkHuy9zh2rd.png)

![浮点数运算指令](https://i.loli.net/2020/02/02/bU6mWDzEVqlZ2ro.png)

其中，为了加速音视频等运算，浮点数的`addps`指令允许多个浮点数并行运算。

### C语句对应的汇编代码

#### 工具

根据机器系统和编译选项设置的差异，编译器会对C语言源代码进行不同程度的优化，所以在每台机器编译得到的汇编代码都会有所不同。利用gcc，我们通过下面的命令来生成**未优化**的汇编代码：

```
gcc -Og xxx.c -S xxx.a
```

由于汇编代码和机器码是一一对应的关系。我们可以通过下面的命令将目标文件或可执行文件(.o/.out)的机器码反汇编(Disassemble)为汇编代码：

```
objdump -d xxx.o
```

第三种读汇编代码的方式是通过调试器gdb，调试文件需要在编译时需要设置调试选项：

```
gdb xxx.out
disassemble <function_name>/...
```

更多的gdb教程可以参考[Quick Guide to GDB](http://beej.us/guide/bggdb/)，以及[Cheat Sheet](http://csapp.cs.cmu.edu/3e/docs/gdbnotes-x86-64.pdf)

#### 条件控制(Conditional Jumping)

条件控制利用测试语句和跳转语句(Jumping)实现。测试语句不保存计算结果，只影响状态位。测试语句有：

* `cmpq src2, src1`：计算src1-src2

* `testq src2, src1`：计算src1&src2

跳转语句根据使用的状态码分为不同的版本：

![x86-64跳转语句](https://i.loli.net/2020/02/02/qYU5F7H8Zwyb6GV.png)

可以使用goto语句来表示跳转的逻辑：

```
=ntest = !Test;
=if (ntest) goto Else;
=// then ...
=goto Done;
Else:
// else ...
Done:
// ...
```

#### 条件转移(Conditional Move)

C语言中if-else语句和?:表达式还可以对应一种叫作**条件转移**的汇编实现。

这种实现提前计算好每个分支的值，然后根据测试值才决定采用哪一个值。

```
result = Then_Expr;
eval = Else_Expr;
nt = !Test;
if (nt) result = eval;
return result;
```

gcc在遇到if-else语句时会在**条件控制**和**条件转移**中二选一。由于现代处理器的流水线设计，条件控制结构的效率更高。

但是下面的代码会造成问题：

* `val = Test(x) ? Hard1(x) : Hard2(x)`: 在分支选择前会运行Hard1(x)和Hard2(x)，可能消耗大量性能；

* `val = p ? *p : 0`: 尽管p为空指针，也会计算`*p`的值，导致段错误(Segmentation Fault)；

* `val = x > 0 ? x*=7 : x+=3`: 两个有副作用的表达式都会执行，而且结果无法预测，因为执行顺序未知。


#### 循环语句

* `do...while`语句，这是汇编实现最简单的循环结构，只需要在循环体尾部增加测试语句：

```
loop:
  // ...Body
  if (Test)
    goto loop
```

* `while`语句，有两种方式，一是进入循环时即跳转到尾部的测试部分，二是基于`do...while`语句实现，但在进入循环时先进行一次测试：

```
  goto test;
loop:
  // ...Body
test:
  if (Test) goto loop;
done:
```

```
  if (!Test) goto done;
  do {
      // ... Body
  } while (Test);
done:
```

* `for`语句，在`while`语句的基础上，开头添加初始化(Init)的代码，循环体后添加更新(Update)代码：

```
// ...Init
while (Test) {
    // ...Body
    // ...Update
}
```

#### switch语句

下面代码使用了C语言switch语句multiple case labels, falling through cases和default case：

```
int w = 1;
switch(x) {
    case 1:
        w = y*z;
        break;
    case 2:
        w = y/z;
        /* Fall Through */
    case 3:
        w += z;
        break;
    case 5:
    case 6:
        w -= z;
        break;
    default:
        w = 2;
}
```

switch语句通过一种叫作**跳转表(Jump Table)**的数据结构来实现这多种特性。跳转表实际上是一个数组，x的取值作为下标，各个case代码的地址作为元素。可以用下面的语句理解跳转表的跳转过程（但C语言不支持这样的语句）：

```
goto *JTab[x];
```

汇编这样实现跳转过程：

```
switch_eg:
    movq    %rdx, %rcx
    cmpq    $6, %rdi   # x:6
    ja      .L8
    jmp     *.L4(,%rdi,8)
```

%rdi存储了x的值，首先先判断x的大小，如果超过跳转表能处理的范围（小于0或大于6）则直接跳转到标签.L8，即default对应的代码处。而标签.L4处存储了跳转表，下面是跳转表的实现：

```
.L4:
	.quad	.L8	# x = 0
	.quad	.L3	# x = 1
	.quad	.L5	# x = 2
	.quad	.L9	# x = 3
	.quad	.L8	# x = 4
	.quad	.L7	# x = 5
	.quad	.L7	# x = 6
```

可以看到x=5和x=6时都跳转到标签.L7，跳转表映射到同一个值，这样就实现了multiple case labels。而falling through利用了新增的merge代码块，作为多个标签的共享代码，实现思路如下：

```
case 2:
    w = y/z;
    goto merge;
case 3:
    w = 1;
merge:
    w += z;
```

对应的汇编代码如下：

```
.L5:                  # Case 2
   movq    %rsi, %rax
   cqto
   idivq   %rcx       #  y/z
   jmp     .L6        #  goto merge
.L9:                  # Case 3
   movl    $1, %eax   #  w = 1
.L6:                  # merge:
   addq    %rcx, %rax #  w += z
   ret
```

当分支值跨度很大时，比如只有1和1000000两个分支，这时跳转表要使用1000000个条目来处理。gcc还是回到条件控制的实现方式生成汇编代码，于是无法一蹴而就地跳转到对应的分支，但可以利用类似二分查找来加速分支查找。

