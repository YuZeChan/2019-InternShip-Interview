​	定义一个类一般都是在头文件中进行类声明，在cpp文件中实现，但使用模板时应注意目前的C++编译器还无法分离编译，须将实现代码和声明代码均放在头文件中。

​	**这块还得明确一个问题，就是template在实例化之前不会编译成二进制代码。**

​	**这里说的是只在template.cpp文件中#include template.h文件的情况，以及在main.cpp中只#include .h文件，并非#include template.cpp（当然这么做，会报重复定义的问题）。**

------



##### 《C++编程思想》第15章(第300页)说明了原因：

模板定义很特殊。由template<…> 处理的任何东西都意味着编译器在当时不为它分配存储空间，它一直处于等待状态直到被一个模板实例告知。在编译器和连接器的某一处，有一机制能去掉指定模板的多重定义。所以为了容易使用，几乎总是在头文件中放置全部的模板声明和定义。



​	至于为什么几个大佬写的已经十分清楚了：

<https://blog.csdn.net/wly_2014/article/details/50575624>

<https://blog.csdn.net/bichenggui/article/details/4207084>

------

**.h文件称为头文件，.cpp文件称为源文件。**

编译器编译时是以一个**源文件作为单元**编译的，当它遇到不在本文件中定义的函数时，若能够找到其声明，则会将此符号放在本编译单元的**外部符号表**中，链接的时候自然就可以找到该符号的定义了。



​	首先，一个编译单元（translation unit）是指一个.cpp文件以及它所#include的所有.h文件，**.h文件里的代码将会被扩展到包含它的.cpp文件里**，然后编译器编译该.cpp文件为一个.obj文件（假定我们的平台是win32），后者拥有PE（Portable Executable，即windows可执行文件）文件格式，并且本身包含的就已经是二进制码，但是不一定能够执行，因为并不保证其中一定有main函数。当编译器将一个工程里的所有.cpp文件以**分离的方式编译**完毕后，再由连接器（linker）进行连接成为一个.exe文件。

​	如果main函数调用了其他cpp中的函数（假设为f），在生成的.obj文件中不会生成有关f函数的二进制代码，只会生成一行call指令。但main函数中没有实现f的代码，就需要连接器的起作用了。连接器负责在其它的.obj中寻找f的实现代码，找到以后将call f这个指令的调用地址换成实际的f的函数进入点地址。**实际上也就是一个jmp 0xABCDEF**。

​	但是，连接器是**如何找到f的实际地址**的呢（在本例中这处于test.obj中），因为.obj与.exe的格式是一样的，在这样的文件中有一个符号导入表和符号导出表（import table和export table）其中将所有符号和它们的地址关联起来。这样连接器只要在f实现的.obj的符号导出表中寻找符号f（当然C++对f作了mangling）的地址就行了，然后作一些偏移量处理后（因为是将两个.obj文件合并，当然地址会有一定的偏移，这个连接器清楚）写入main.obj中的符号导入表中f所占有的那一项即可。



​	说到底是**分离编译的一种弊端**，加上类模板的特点：**无法在类模板的对象实例化之前编译**（只有知道数据类型后，经过计算，才能在内存中生成相应的二进制代码）。



​	1.如果类模板的.h和.cpp文件分开，即.h文件没有类成员函数的实现。在include其中的.h文件后，.h文件相当于整体复制到main.cpp文件中，但在调用实例化的f的过程中，只见到f的声明（当编译器只看到模板的声明时，它不能实例化该模板，**只能创建一个具有外部连接的符号并期待连接器能够将符号的地址决议出来**），无法找到f的定义，则只能寄希望于**连接器**在.obj文件中能够发现f的实现。

​	2.而在类模板的.cpp文件中，并没用实例化类模板的对象，所以在分离编译时（编译器编译某一个.cpp文件时并不知道另一个.cpp文件的存在，也不会去查找（当遇到未决符号时它会寄希望于连接器）），类模板的.cpp文件所生成的.obj文件依然不会生成关于类模板成员函数的二进制代码。所以连接器也黔驴技穷了，在所有的.obj文件中仍然找不到f的实现代码。链接就会出现f函数未定义的错误。