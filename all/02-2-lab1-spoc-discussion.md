# lec4: lab1 SPOC思考题

##**提前准备**
（请在上课前完成）

 - 完成lec4的视频学习和提交对应的在线练习
 - git pull ucore_os_lab, v9_cpu, os_course_spoc_exercises in github repos。这样可以在本机上完成课堂练习。
 - 了解x86的保护模式，段选择子，全局描述符，全局描述符表，中断描述符表等概念，以及如何读写，设置等操作
 - 了解Linux中的ELF执行文件格式
 - 了解外设:串口，并口，时钟，键盘,CGA，以及如何对这些外设进行编程
 - 了解x86架构中的mem地址空间和io地址空间
 - 了解x86的中断处理过程（包括硬件部分和软件部分）
 - 了解GCC的x86/RV内联汇编
 - 了解C语言的可函数变参数编程
 - 了解qemu的启动参数的含义
 - 在piazza上就lec3学习中不理解问题进行提问
 - 学会使用 qemu
 - 在linux系统中，看看 /proc/cpuinfo的内容

## 思考题

### 启动顺序

1. x86段寄存器的字段含义和功能有哪些？

   常见寄存器：

   + CS：代码段寄存器
   + DS：数据段寄存器
   + SS：堆栈段寄存器

   **字段含义**：段寄存器的高13位[3-15bit]是Index，用以在GDT中查找表项。第2位是Table Indicator，0表示为全局描述表(GDT)，1表示局部描述表(LDT)，ucore中只需要使用GDT。低两位{0-1bit]是请求优先级(RPL)，可以是0、1、2、3四个特权级，操作系统的特权级最高，为0，应用程序放在2这个特权级。

   **功能**：段选择器通过GDT Index查找到GDT中的段描述符，获得段基址、段大小等重要信息，段基址base与EIP寄存器中的段偏移量offset相加获得线性地址，在分页机制没用启动时，线性地址就是物理地址。

2. x86描述符特权级DPL、当前特权级CPL和请求特权级RPL的含义是什么？在哪些寄存器中存在这些字段？对应的访问条件是什么？

   **DPL**：储存在段描述符，规定访问该段的权限级别，只有max(CPL,RPL)<=DPL时，进程才能够访问一个段，每个段的DPL固定。

   **CPL**：当前进程的权限级别，是正在执行的代码段的特权级，存在于CS寄存器的最低两位，也就是CS.RPL。

   **RPL**：是进程对段访问的请求权限。每个段选择子都有自己的RPL，它会削弱CPL的作用。

3. 分析可执行文件格式elf的格式（无需回答）

   ```c
   /* file header */
   struct elfhdr {
       uint32_t e_magic;     // must equal ELF_MAGIC
       uint8_t e_elf[12];
       uint16_t e_type;      // 1=relocatable, 2=executable, 3=shared object, 4=core image
       uint16_t e_machine;   // 3=x86, 4=68K, etc.
       uint32_t e_version;   // file version, always 1
       uint32_t e_entry;     // entry point if executable
       uint32_t e_phoff;     // file position of program header or 0
       uint32_t e_shoff;     // file position of section header or 0
       uint32_t e_flags;     // architecture-specific flags, usually 0
       uint16_t e_ehsize;    // size of this elf header
       uint16_t e_phentsize; // size of an entry in program header
       uint16_t e_phnum;     // number of entries in program header or 0
       uint16_t e_shentsize; // size of an entry in section header
       uint16_t e_shnum;     // number of entries in section header or 0
       uint16_t e_shstrndx;  // section number that contains section name strings
   };
   ```

### 4.1 C函数调用的实现

### 4.2 x86中断处理过程

1. x86/RV中断处理中硬件压栈内容？用户态中断和内核态中断的硬件压栈有什么不同？

   **内核态**：还在内核态执行中断例程，在内核态的栈里压入Error Code，EIP，CS，EFLAGS。中断例程代码执行完成后继续在内核态执行。

   **用户态**：由用户态转到内核态执行中断例程，除了在内核态的栈里压入Error Code，EIP，CS，EFLAGS以外，还要把用户态栈的信息ESP，SS压入内核态的栈中。中断例程代码执行完成后返回在用户态执行。

2. 为什么在用户态的中断响应要使用内核堆栈？

   因为用户态发生中断后，会进入内核态执行中断服务里程，需要保护好内核态原本正在执行的程序。并且也为了保护内核态的安全。

3. x86中trap类型的中断门与interrupt类型的中断门有啥设置上的差别？如果在设置中断门上不做区分，会有什么可能的后果?

   调用Interrupt Gate时，中断会被CPU自动禁止。而调用Trap Gate时，CPU则不会禁止或打开中断，而是保留它原来的样子。

   如果在设置上不做区分，会导致重复触发中断。嵌套中断，指的是而是先关掉中断，处理完当前中断之后再顺序处理下一个。

### 4.3 练习四和五 ucore内核映像加载和函数调用栈分析

1. ucore中，在kdebug.c文件中用到的函数`read_ebp`是内联的，而函数`read_eip`不是内联的。为什么要设计成这样？

   ebp的值可以直接通过命令获得，如果不设置为inline的，就会在函数调用过程中修改掉ebp本身的值。

   eip的值不可以直接通过命令获得，具体访问方式利用了call命令会将eip压栈，从而使用read_eip函数读取栈上的eip值，如果设计为inline的，那没有函数调动也就不会把eip压栈。

   ```C
   static __noinline uint32_t
   read_eip(void) {
       uint32_t eip;
       asm volatile("movl 4(%%ebp), %0" : "=r" (eip));
       return eip;
   }
   ```

### 4.4 练习六 完善中断初始化和处理

1. CPU加电初始化后中断是使能的吗？为什么？

   不是的，此时处于实模式，且中断机制还没有建立好。BIOS会在POST自检完成后在内存中建立中断向量表和中断服务程序，完成整个中断机制的建立后，才会enable。

## 开放思考题

1. 在ucore/rcore中如何修改lab1, 实现在出现除零异常时显示一个字符串的异常服务例程？
2. 在ucore lab1/bin目录下，通过`objcopy -O binary kernel kernel.bin`可以把elf格式的ucore kernel转变成体积更小巧的binary格式的ucore kernel。为此，需要如何修改lab1的bootloader, 能够实现正确加载binary格式的ucore OS？ (hard)
3. GRUB是一个通用的x86 bootloader，被用于加载多种操作系统。如果放弃lab1的bootloader，采用GRUB来加载ucore OS，请问需要如何修改lab1, 能够实现此需求？ (hard)
4. 如果没有中断，操作系统设计会有哪些问题或困难？在这种情况下，能否完成对外设驱动和对进程的切换等操作系统核心功能？

## 课堂实践
### 练习一
在Linux系统的应用程序中写一个函数print_stackframe()，用于获取当前位置的函数调用栈信息。实现如下一种或多种功能：函数入口地址、函数名信息、参数调用参数信息、返回值信息。

### 练习二
在ucore/rcore内核中写一个函数print_stackframe()，用于获取当前位置的函数调用栈信息。实现如下一种或多种功能：函数入口地址、函数名信息、参数调用参数信息、返回值信息。
