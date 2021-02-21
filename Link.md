<h1>Link</h1>

<h2>编译</h2>

​		C语言代码到机器可执行程序的过程，分为四个步骤：

`预处理：处理C语言代码中的宏定义指令，条件编译指令，拓展头文件包含指令，将.c文件转化成.i文件`

`编译：将预处理后的C代码转化成汇编代码，语法分析，语义分析在这一步完成`

`汇编：汇编器将汇编代码处理后成为对象程序，其中定义的全局符号（定义在头文件中的函数以及一些全局变量）还没有被链接`

`链接器：对象程序以及所需的所需的静态库或者动态共享库经过链接器的处理最终成为计算机可执行程序`

<strong>例子</strong>

```c
// 文件 main.c
int sum(int *a, int n);

int array[2] = {1, 2};

int main()
{
    int val = sum(array, 2);
    return val;
}

// -----------------------------------------
// 文件 sum.c
int sum(int *a, int n)
{
    int i, s = 0;
    for (i = 0; i < n; i++)
        s += a[i];
    
    return s;
}
```

​		编译命令：

```bash
linux> gcc -Og -o prog main.c sum.c
linux> ./prog
```

![img](https://wdxtub.com/images/14613261957551.jpg)

​		程序编译的大致过程如图所示。

<h2>链接</h2>

<h3>使用链接器的原因：</h3>

1.模块化的角度考虑，可以把程序分散到不同的小的源代码中，可以复用常见的功能和库；

2.效率角度考虑，改动代码时只需要重新编译改动的文件，其他不受影响。常用的函数和功能可以封装成库，提供给程序进行调用；

<h3>链接器所做的事</h3>

<strong>符号解析</strong>

​		我们在代码中会声明变量及函数，之后会调用变量及函数（`主要是全局变量和静态变量,局部变量保存在栈中，并没有生成符号信息`)，所有的符号声明都会被保存在符号表(symbol table)中，而符号表会保存在由汇编器生成的 object 文件中（也就是 `.o` 文件）。符号表实际上是一个结构体数组，每一个元素包含名称、大小和符号的位置。

​		在 symbol resolution 阶段，链接器会给每个符号应用一个唯一的符号定义，用作寻找对应符号的标志。

<strong>重定位 Relocation</strong>

​		这一步所做的工作是把原先分开的代码和数据片段汇总成一个文件`不同可执行文件中的.text,.bss,.data等段汇聚成一个段`，会把原先在 `.o` 文件中的相对位置转换成在可执行程序的绝对位置，并且据此更新对应的引用符号（才能找到新的位置）

<h3>三种对象文件</h3>

<strong>可重定位目标文件 Relocatable object file</strong>

​		每个`.o`文件都是由对应的`.c`文件通过编译器和汇编器生成，包含代码和数据，可以与其他可重定位目标文件合并创建一个可执行或共享的目标文件；

<strong>可执行目标文件（`a.out`file）</strong>

​		由链接器生成，可以直接通过加载器加载到内存中充当进程执行的文件，包含代码和数据

<strong>共享目标文件（`.so`file）</strong>

​		可以在链接（静态共享库）时加入目标文件或加载时或运行时（动态共享库）

被动态加载到内存并执行。

<h3>ELF文件格式</h3>

![img](https://wdxtub.com/images/14613276806705.jpg)

- `ELF header`
  - 包含 word size, byte ordering, file type (.o, exec, .so), machine type, etc
- `Segment header table`
  - 包含 page size, virtual addresses memory segments(sections), segment sizes
- `.text section`
  - 代码部分
- `.rodata section`
  - 只读数据部分，例如跳转表
- `.data section`
  - 初始化的全局变量
- `.bss section`
  - 未初始化的全局变量
- `.symtab section`
  - 包含 symbol table, procudure 和 static variable names 以及 section names 和 location
- `.rel.txt section`
  - .text section 的重定位信息
- `.rel.data section`
  - .data section 的重定位信息
- `.debug section`
  - 包含 symbolic debugging (`gcc -g`) 的信息
- `Section header table`
  - 每个 section 的大小和偏移量

链接器实际上会处理三种不同的符号，对应于代码中不同写法的部分：

- 全局符号 Global symbols
  - 在当前模块中定义，且可以被其他代码引用的符号，例如非静态 C 函数和非静态全局变量
- 外部符号 External symbols
  - 同样是全局符号，但是是在其他模块（也就是其他的源代码）中定义的，但是可以在当前模块中引用
- 本地符号 Local symbols
  - 在当前模块中定义，只能被当前模块引用的符号，例如静态函数和静态全局变量
  - 注意，Local linker symbol 并不是 local program variables