---
title: CMU 15-213 Part7 Linking
date: "2020/2/9"
categories:
- CMU 15-213
---

### 可执行文件的结构

编译器/链接器可以输出三种目标文件，这三种目标文件都是二进制文件：

* 可重定位的目标文件(.o等)，由编译器输出，无法加载到内存或运行，需要链接器进一步处理；

* 可执行的目标文件(.out, .exe等)，由链接器输出，可以加载到内存或执行

* 库目标文件(.so等)，动态共享库

目标文件都采用ELF格式编码(Excutable and Linkable Format)

![ELF编码](https://i.loli.net/2020/02/08/ogAyFqJG8DIbRC6.png)

* ELF Header：定义字长，字节序，文件类型(.o, .out, .so等)，机器类型等信息；

* Segment header table：只有可执行文件必须，定义各个段(共享内存段，代码段，数据段等)在内存中的位置；

* .text：代码段；

* .rodata：只读数据段，存储只读数据，如switch语句的跳转表等；

* .data：**已初始化**的全局变量；

* .bss：**未初始化**的全局变量，相比.data段，只保存符号的信息，并不实际的占用内存；

* .symbol：符号表，函数，全局变量，函数内部的静态变量等；

* .rel：记录需要重定位的符号，链接器生成可执行文件时根据该表修改符号地址；

* .debug：记录调试时需要的行号等信息；

* Section header table：记录各个区段的起始地址。


### 链接

C源代码文件经过预处理、编译、链接等步骤才产生可执行文件。编译生成的是目标文件(.o文件)，而链接将多个用户的.o文件与静态库中的.o文件整合到一起，才生成可执行文件。链接可以提高程序的模块化和编译的效率。正因为链接这个步骤存在，才可能存在程序库，同时我们也可以把代码分开放到不同的源文件中；而且修改后也只需要重新编译变更的源文件，其他目标文件无需更改。

* 符号解析(Symbol Resolution)

全局的函数和变量在链接器中称为符号。如果遇到全局函数或变量的定义，会将它们添加到**符号表**中。符号表存储了符号的名称，大小和位置等信息。对全局函数的调用，或对全局变量的引用称作符号的引用。符号引用往往与定义不在同一个文件中，所以编译器生成目标文件(.o文件)时不能确定这些符号定义的位置。编译阶段结束后，链接器在符号解析步骤中会将这些引用唯一的对应符号表中一个符号。

* 重定位(Rolocation)

在重定位阶段，链接器已经将多个模块(.o文件，库文件)整合到一个可执行文件中，可以加载到系统中或执行。在重定位前，所有的符号都只有它在各自模块中的偏移量，而无法确定被加载到内存的位置。所以，在整合完成后，重定位把所有符号重新定义在真实的内存地址。同时，对于所有的符号引用，也将引用地址修改为内存中的绝对地址。

![可重定位目标文件](https://i.loli.net/2020/02/08/Y3hE2BNGsV5xayz.png)

上图是编译器生成的一个可重定位目标文件，编译器把所有不能确定的符号引用都写作0，而红色标记的文本是起标记作用的重定位条目，`a`和`f`是链接器将修改的相对地址，`R_X86_64_32`和`R_X86_64_PC32`定义了引用的数据类型，前者是x86_64架构32位的数据，后者是32位的指针，最后的`array`和`sum`是符号名称。`sum-0x4`是对当前PC值的偏移，因为`sum`返回`sum`到`main+f`的相对距离，而此时PC指向下一条指令处。

![整合目标文件](https://i.loli.net/2020/02/08/F947mPe2dpATYX1.png)

![可执行文件加载到内存](https://i.loli.net/2020/02/08/xOeNvlhnzEFGfBk.png)

链接器将多个目标文件整合为一个可执行的目标文件后，该可执行目标文件可以直接把内容加载到内存中并开始运行。.rodata，.text加载到内存的只读部分，.data，.bss加载到内存的数据部分。

相对一个模块m来说，所有出现的符号可以分为3种类型：全局符号(Global Symbol)，外部符号(External Symbol)，本地符号(Local Symbol)

* 全局符号：模块m定义的外部可见的符号；

* 外部符号：模块m引用的定义在其他模块的符号，等同与用`extern`修饰的外部变量；

* 本地符号：模块m定义的，但只在模块m内部可见的符号，比如用static修饰的全局变量/函数（注意与函数内的本地变量区分）。

注：函数内部的static变量是特殊的本地符号，也存储在.data段中，拥有全局生命周期，但是只在函数内可见。

链接器根据是否初始化，把全局变量分为强变量和弱变量。可以在多个模块声明多个同名的全局变量，但是不允许存在两个同名的强变量。如果多个模块中存在一个强变量，多个弱变量，链接器选取强变量作为定义，其他弱变量被重定位到强变量的地址，等同于用`extern`声明。但是如果只存在多个弱变量，链接器将随机选择一个作为定义，结果可能导致不可预测的情况，比如下面这个例子

```cpp
// m1.c
int x;
int y;
void p1 {}
```

```cpp
// m2.c
double x;
void p2 {}
```

在这个例子中，m1.c和m2.c定义的x虽然类型不同，但是链接器识别为同一个符号，将随机采取一个作为定义。如果链接器采取m1.c中的x作为定义，那么m2.c的x将作为外部声明。但是m2.c把x解释为double类型，在m2.c写入x将写入8个字节，会导致y被覆盖。(因为写入x的代码在编译m2.c时已经确定)

```cpp
// m1.c
int x=0;
int y;
void p1 {}
```

```cpp
// m2.c
double x;
void p2 {}
```

这个例子也会导致相同的问题。解决办法是总是初始化全局变量，总是使用`static`和`extern`关键字来明确全局变量的作用域和来源。

### 库

库可以分为静态库和动态库。

* 静态库(.a文件)，.o文件的集合，每个.o文件都对应一个函数，比如libc.a是printf.o, malloc.o等文件的集合。链接器只链接用户使用的.o。

![制作静态库](https://i.loli.net/2020/02/08/YK4GroOu53ChdzQ.png)

用户通过`ar`命令手动打包.o文件来制作静态库。若使用静态库，用户则需要设置gcc编译选项`-l<name>`。例如`-lm`和`-lvector`会链接系统中的libm.a和当前目录下的libvector.a(自动补全前缀lib和后缀.a)。注意，作为参数的文件顺序会影响链接。链接器按照传参顺序处理文件，并维护一个未定义的引用列表。每遇到一个新文件，会更新该列表。如果所有文件都处理完，列表中还存在未定义的引用列表，则报错。所以被引用的库文件要作为后面的参数传入。

```shell
unix> gcc -L. test.o -lvector 
unix> gcc -L. -lvector test.o 
test.o: In function `main': 
test.o(.text+0x4): undefined reference to `libvector' 
```

![引入静态库](https://i.loli.net/2020/02/08/qOp8laFwJZEK9hR.png)

静态库的缺点在于每个程序都包含一份库的副本，程序体积变大，而且库更改后要重新链接。

* 动态共享库(.so, .ddl)，所有程序在系统中共享一份库的代码副本，程序加载到内存中时才进行链接，甚至在运行时进行链接。

![共享动态库](https://i.loli.net/2020/02/08/jdiIPrg562VG47p.png)

用户需要开启gcc编译器的`-shared`选项来制作动态共享库。若使用动态共享库，用户仍需传入.so库文件，但是库文件只提供符号信息，代码并不会整合到可执行文件。代码运行时，系统通过动态链接器(dynamic linker, dl-linux.so)再重新找到需要的.so库文件，加载代码和数据。

另一种动态链接的方法是运行时链接，程序可以在运行时调用`dlopen`，`dlsym`函数获取库函数的接口。


### 库打桩技术(Interpositioning)

库打桩技术可以分别在编译，链接，或动态链接时拦截函数调用，插入用于调试，记录，测试等代码，实现类似装饰器的效果。下面以为`malloc`和`free`函数增加调试信息输出代码为例子，演示三种库打桩方法。

```cpp
#include "malloc.h"

int main() {
    int *p = malloc(32);
    free(p);
    return 0;
}
```

#### 编译时(Compile Time)库打桩

```cpp
// malloc.c
#ifdef COMPILETIME
#include <stdio.h>
#include <malloc.h>

void *mymalloc(size_t size)
{
    void *ptr = malloc(size);
    printf("malloc(%d)=%p\n",
           (int)size, ptr);
    return ptr;
}

void myfree(void *ptr)
{
    free(ptr);
    printf("free(%p)\n", ptr);
}
#endif
```

```cpp
// malloc.h
#define malloc mymalloc
#define free myfree
void *mymalloc(size_t size);
void free(void *ptr);
```

```shell
gcc -Wall -DCOMPILETIME -c mymalloc.c
gcc -Wall -I. -o manc main.c mymalloc.o
```

编译时库打桩使用`#define`覆盖函数调用。在main.c包含malloc.h时，`#define`就会把`malloc`和`free`的调用替换为`mymalloc`和`myfree`。编译命令中`-D`选项设置定义宏，`-I.`添加当前目录到包含目录，即搜索头文件(malloc.h)时优先搜索当前目录，否则`#include <malloc.h>`不会在当前目录搜索。(MacOS的`malloc`函数声明在malloc/_malloc.h文件中，所以我们自定义的malloc.h要更名为_malloc.h放在malloc文件夹中)


#### 链接时(Link-time)库打桩

```cpp
#ifdef LINKTIME
#include <stdio.h>

void *__real_malloc(size_t size);
void __real_free(void *ptr);

/* malloc wrapper function */
void *__wrap_malloc(size_t size)
{
    void *ptr = __real_malloc(size); /* Call libc malloc */
    printf("malloc(%d) = %p\n", (int)size, ptr);
    return ptr;
}

/* free wrapper function */
void __wrap_free(void *ptr)
{
    __real_free(ptr); /* Call libc free */
    printf("free(%p)\n", ptr);
}
#endif
```

```shell
gcc -Wall -DLINKTIME -c mymalloc.c
gcc -Wall -c main.c
gcc -Wall -Wl,--wrap,malloc -Wl,--wrap,free -o mainl int.o mymalloc.o
```


链接时库打桩利用连接器(ld)的别名选项`--wrap=<symbol>`。设置该选项后，所有`<symbol>`的未定位引用都定位到`__wrap_<symbol>`，而所有`__real_<symbol>`都定位到`<symbol>`。而在编译器gcc中，跟在`-Wl,`后面的字符串会传递给链接器作为参数，只不过将逗号替换为空格。(MacOS下`ld`无`--wrap`选项，无法实现链接时打桩。可参考[StackOverflow](https://stackoverflow.com/questions/39441587/wrapping-symbols-during-linking-on-os-x))


#### 动态加载库打桩

```cpp
#ifdef RUNTIME
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <dlfcn.h>

void *malloc(size_t size)
{
    void *(*mallocp)(size_t size);
    char *error;

    mallocp = dlsym(RTLD_NEXT, "malloc"); /* Get addr of libc malloc */
    if ((error = dlerror()) != NULL) {
        fputs(error, stderr);
        exit(1);
    }
    char *ptr = mallocp(size); /* Call libc malloc */
    printf("malloc(%d) = %p\n", (int)size, ptr);
    return ptr;
}

void free(void *ptr)
{
    void (*freep)(void *) = NULL;
    char *error;

    if (!ptr)
        return;

    freep = dlsym(RTLD_NEXT, "free"); /* Get address of libc free */
    if ((error = dlerror()) != NULL) {
        fputs(error, stderr);
        exit(1);
    }
    freep(ptr); /* Call libc free */
    printf("free(%p)\n", ptr);
}
#endif
```

```shell
gcc -Wall -DRUNTIME -shared -fpic -o mymalloc.so mymalloc.c -ldl
gcc -Wall -o mainr main.c
LD_PRELOAD="./mymalloc.so" ./mainr
```

动态加载库打桩甚至不需要原函数的目标文件(.o文件)。将`RTLD_NEXT`作为参数，`dlsym`函数寻找下一个可用的引用。在编译时，使用`-shared`开启生成动态库选项，`-fpic`也是动态库相关选项。执行程序前的`LD_PRELOAD="./mymalloc.so"`可以令程序在该环境变量下运行，`LD_PRELOAD`令程序优先加载其中的动态库，从而实现拦截效果。
