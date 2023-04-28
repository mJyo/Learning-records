# C++编程总结

错误：VS2022报错E0728:不允许使用命名空间名

在使用宏替换时遇到该问题

```c++
#define HZ_BIND_EVENT_FN(fn) std::bind(&fn,this,std::placeholders::_1) //1
dispatcher.Dispatch<MouseButtonPressedEvent (HZ_BIND_EVENT_FN(ImGuiLayer::OnMouseButtonPressedEvent)); //2
```

在为构建之前第一句和第二句产生报错冲突，原因暂不明确

## 链接技术

符号表条目：由汇编器构造，使用编译器输出到汇编语言.s文件中的符号。分外部、内部两种符号类型，局部不在符号表条目中，

```c
typedef struct { 
    int name; //String table offset
    char type:4, //Function or data (4 bits)
    binding:4; //Local or global (4 bits)
    char reserved; //Unused
    short section; //Section header index
    long value; //Section offset or absolute address
    long size; //Object size in bytes
} Elf64_Syrnbol;
```

**ELF符号表条目**：包含一个条目的数组

每个符号被分配到目标文件的某个节中，由section字段表示

三个特殊伪节（pseudosection)：在头节部表中没有条目；ABS代表不该被重定位的符号；

UNDEF代表未定义的符号，本模块中引用，其他地方定义符号；COMMON表示未被分配位置的未初始化的数据目标，即可表示未初始化的全局变量。

## 符号解析

注：C++通过重整(mangling)技术实现函数重载，重整即**编译器**将每个唯一的方法和参数列表组合编码成一个对**链接器**来说唯一的名字

函数和一初始化的全局变量是**强符号**，未初始化的全局变量是**弱符号**

Linux链接器处理多重定义符号名：

- 规则1：不允许有多个同名的强符号
- 规则2：如果有一个强符号和弱符号同名，选择强符号
- 规则3：如果有多个弱符号同名，从这些若符号任选一个

##### 与静态库链接

静态库：做连接器的输入，当链接器构造一个输出的可执行文件时，它只复制静态库里被程序**引用**的目标模块

历史背景：平衡编译器开发复杂度和应用程序员调用标准库函数难度

三种历史解决方案

- 方案1：让编译器辨识出对标准库函数的调用，并直接生成相应代码，但仅适合**小型**的库函数，应用案例有Pascal语言,对应用程序员极其方便
- 方案2：将标准库的函数都放在一个单独的可重定位目标模块中，应用程序员可通过这个模块直接链接到可执行程序中，缺点点是每个可执行文件中都有一份标准库函数的**完全拷贝**，极大浪费存储空间。且要求库的开发人员再库更改时重新编译整个源文件，使得标准库函数的开发与维护变得复杂
- 方案3：为每个库函数创建一个独立的可重定位文件，要求应用程序员**显式链接**目标模块，过程耗时易出错

Linux下使用AR工具创建函数的静态库：

```shell
gcc -c addvector.c mulvector.c
ar rcs libvector.a add.o mul.o 
```

通过编译链接输入文件main2.o和libvector.a

-static参数告诉编译器驱动程序，链接器应链接一个完全链接的可执行目标文件，可加载到内存运行

```shell
gcc -c main2.c
gcc -static -o prog2c main2.o ./libvector.a
```

![image-20230420120604264](C:\Users\SEMI\AppData\Roaming\Typora\typora-user-images\image-20230420120604264.png)

链接器运行时，判定main2.o引用了addvector.o定义的advector符号，复制addvec.o到可执行文件

##### 使用静态库解析引用

链接器维护一个可重定位目标集合E，未解析符号引用集合U，在前面输入文件中已定义文件集合D。

初始时，三个集合均为空，链接器判断输入文件f，重复执行输入文件队列，直到U，D不再发生变化，若完成对命令行中输入文件的扫描后，U非空，链接器输出错误并终止

- f是目标文件，将f添加到E,修改U，D反映f中的符号定义与引用
- f是存档文件，链接器尝试匹配U中未解析的符号和存档文件成员定义的符号，任何不包含在E中的成员目标文件予以丢弃。如存档文件成员m,定义一个符号解析U中的的一个引用，将m加到E中，修改U，D反映符号定义和引用

注：考虑如下命令

```shell
gcc -static ./libvector.a main2.c
```

在处理libvector.a 时，U为空，所以没有libvector.a中的成员目标文件会添加到E中，对addvec的引用就无法解析。故一般准则是将库放在目标模块后

### 重定位

时机：完成符号解析之后，链接器获取到了目标模块中代码节和数据节的确切大小，开始进行重定位

目标：合并输入模块，为每个符号分配运行时地址

- 重定位节和符号定义：将所有同类型的节合并为新的聚合节。如.data节被全部合并成一个节，新的聚合节成为可执行目标文件的.data节。链接器将运行时链接地址赋予新的聚合节，完成时程序中的每条指令和全局变量获得了唯一的运行时内存地址
- 重定位节中的符号引用：链接器修改代码节中和数据节中对每个符号的引用，使得他们指向正确的运行时地址。需要依赖可重定位目标模块的的**重定位目标条目(relocation entry)**的数据结构

结合操作系统后续学习...

## 可执行目标文件



## 动态链接共享库

引入背景：

- 静态库需要定期维护和更新，程序员需要去更新库，显式链接更新的库
- C程序的I/O函数会被复制到每个运行进程的文本段中，对“稀缺”内存造成极大浪费

共享库(shared Library)是致力于解决静态库的缺陷的现代产物，为一个目标模块，可以加载到内存的任意位置，由动态链接器(dynamic linker)的程序执行动态链接(dynamic linking)到目标程序

Linux中以.so表示，Windows中以.dll表示

![image-20230420151253759](../AppData/Roaming/Typora/typora-user-images/image-20230420151253759.png)

```shell
gcc -o prog21 main2.c ./libvector.so
```

链接器只需复制一些重定位和符号表信息，他们使得运行时可以解析libvector.so中代码和数据的引用

运用实例：分发软件、高性能Web服务器

## 位置无关代码

结合操作系统后续学习...