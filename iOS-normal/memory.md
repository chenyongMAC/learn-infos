## memory

栈区(stack) 
由编译器自动分配并释放，存放函数的参数值，局部变量等。栈是系统数据结构，对应线程/进程是唯一的。优点是快速高效，缺点时有限制，数据不灵活。［先进后出］
栈空间分静态分配和动态分配两种。
静态分配：编译器完成，如自动变量(auto)的分配
动态分配：alloca函数，自动释放，不鼓励这么做



堆区(heap) 
由程序员分配和释放，如果程序员不释放，程序结束时，可能会由操作系统回收 ，比如在ios 中 alloc 都是存放在堆中。



全局区(静态区) (static)
全局变量和静态变量的存储是放在一起的，初始化的全局变量和静态变量存放在一块区域，未初始化的全局变量和静态变量在相邻的另一块区域，程序结束后有系统释放。
全局区可分为2种：
.bss: 未初始化全局区
data: 初始化全局区



文字常量区
存放常量字符串，程序结束后由系统释放


代码区
存放函数的二进制代码




## Apply for memory
栈：
存储每一个函数在执行的时候都会向操作系统索要资源，栈区就是函数运行时的内存，栈区中的变量由编译器负责分配和释放，内存随着函数的运行分配，随着函数的结束而释放，由系统自动完成。
栈是向低地址扩展的数据结构，是一块连续的内存的区域。如果申请的空间超过栈的剩余空间时，将提示overflow。因此，能从栈获得的空间较小。


堆：
1.首先应该知道操作系统有一个记录空闲内存地址的链表。
2.当系统收到程序的申请时，会遍历该链表，寻找第一个空间大于所申请空间的堆结点，然后将该结点从空闲结点链表中删除，并将该结点的空间分配给程序。
3.由于找到的堆结点的大小不一定正好等于申请的大小，系统会自动的将多余的那部分重新放入空闲链表中
堆是向高地址扩展的数据结构，是不连续的内存区域。这是由于系统是用链表来存储的空闲内存地址的，自然是不连续的，而链表的遍历方向是由低地址向高地址。堆的大小受限于计算机系统中有效的虚拟内存。



