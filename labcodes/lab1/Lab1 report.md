# Lab1 report

## 练习1：理解通过make生成执行文件的过程。（要求在报告中写出对下述问题的回答）

### 1.操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中每一条相关命令和命令参数的含义,以及说明命令导致的结果）

下面是生成ucore.img的代码

```makefile
# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
```

`UCOREIMG	:= $(call totarget,ucore.img)`

作用：定义变量，即用UCOREIMG这个变量表示$(call totarget,ucore.img)；

`$(UCOREIMG): $(kernel) $(bootblock)`

这句话代表生成的主体规则，分别是目标：前置条件，**所以ucore.img生成需要用到两个文件，分别就是$(kernel)和$(bootblock)**

`$(V)dd if=/dev/zero of=$@ count=10000`

就是将/dev/zero下面的块文件拷贝到$@这个地方：/dev/zero代表的是无数多的0，一定比你需要的数目多，而$@代表的是$(UCOREIMG），可以看做ucore在内存中的起始地址，最后的 count=10000表示需要拷贝的大小为10000block，是足够ucore使用的。**所以这一句相当于初始化ucore所需的地址空间。**

`$(V)dd if=$(bootblock) of=$@ conv=notrunc`

将$(bootblock)的内容拷贝到ucore所在的内存空间，且选项conv=notrunc的意思是表示不截短输出文件地输出；

`$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc`

相当于将$(kernel)的内容拷贝到ucore所在的内存空间，且选项内容是不截短输出文件地输出以及从输出文件开头跳过1个块后再开始复制。跳过一个块是为了避免覆盖掉$(bootblock)的内容

**总的来说ucore.img的生成分为三个步骤，第一是初始化ucore所需的地址空间，第二将bootblock复制到ucore的第一个块中；第三将kernel复制到ucore第二个块开始的空间中。**

这里我们需要用到bootblock和kernel，后面将解释这两部分的生成

#### 1.1 bootblock的生成：

如下是生成bootblock的代码

```makefile
# create bootblock
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJDUMP) -t $(call objfile,bootblock) | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)
```

前面三条指令主要是变量的相关定义，bootfiles主要调用了两个函数关系，实现了将boot目录下的文件加上了.c或者.S的后缀

第二条makefile的功能是将boot下的文件与GCC的编译模版规则进行比对，查看到底是否有语法错误

而第三条是将bin 这个前缀加到boot目录文件前参与编译工作。

接着是主要部分：

`$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)`

bootblock可以由两种方式生成，前一个的作用是调用了toobj函数，实现了在.o文件前加obj 前缀的功能，而第二个是在依赖文件sign前加bin。

生成规则的第一条是输出链接文件，第二条是准备将处理后的bootblock加载到内存地址0x7c00处开始运行

这里就是负责加载bootloader的功能部分了，接下去的指令分别代表了汇编和链接功能生成可执行的二进制执行文件

而后调用了create_target函数生成了bootblock。按照操作系统的原理解释，这个bootblock的大小应该是小于512byte的

其中相关参数的含义为：

- ggdb 生成可供gdb使用的调试信息
- -m32生成适用于32位环境的代码
- -gstabs 生成stabs格式的调试信息
- -nostdinc 不使用标准库
- -fno-stack-protector 不生成用于检测缓冲区溢出的代码
- -0s 位减小代码长度进行优化

#### 1.2 kernel的生成

生成kernel的主要代码如下

```makefile
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
    @echo + ld $@
    $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
    @$(OBJDUMP) -S $@ > $(call asmfile,kernel)
    @$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```

结合前面的对kernel的一些声明:

```makefile
# kernel

KINCLUDE    += kern/debug/ \
               kern/driver/ \
               kern/trap/ \
               kern/mm/

KSRCDIR     += kern/init \
               kern/libs \
               kern/debug \
               kern/driver \
               kern/trap \
               kern/mm

KCFLAGS     += $(addprefix -I,$(KINCLUDE))

$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))

KOBJS   = $(call read_packet,kernel libs)
```

这一部分主要应该是包含相应需要用到的头文件，查看文件中的文件得出，生成kernel需要以下文件：kernel.ld    init.o  readline.o  stdio.o     kdebug.o   kmonitor.o panic.o clock.o console.o intr.o  picirq.o   trap.o  trapentry.o  vectors.o  pmm.o  printfmt.o   string.o

根据已有的文件，kernel.ld已经存在，其他的.o文件则需要.c和.s文件通过gcc编译生成

生成init.o需要的命令:

`gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o`

生成其他.o文件的命令和init.o的方法类似，生成这些.o文件后便可以生成kernel可执行文件。得到kernel和bootblock后再如前文所说就可得到ucore.img

### 2.一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

在sign.c中有

```c
if (size != 512) {
        fprintf(stderr, "write '%s' error, size is %d.\n", argv[2], size);
        return -1;
    }

//以及

buf[510] = 0x55;
buf[511] = 0xAA;
```

可知主引导扇区的大小为512字节，而且最后两个字节内容是0x55AA。

## 练习2：使用qemu执行并调试lab1中的软件。（要求在报告中简要写出练习过程）

1. ### 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。

   编辑lab1/tools/gdbinit为:

   ```
   target remote :1234
   set architecture i8086
   ```

   在lab1目录下执行`make debug`，之后使用si命令可使BIOS单步执行

   在gdb中执行`x /2i 0xffff0`可以看到该地址处的第一条指令是`ljmp $0xf0000,$0xe05b`

   接下来可执行`x /10i 0xfe05b`继续跟踪执行过程

2. ### 在初始化位置0x7c00设置实地址断点,测试断点正常。

   编辑lab1/tools/gdbinit为:

   ```
   target remote :1234
   set architecture i8086  
   b *0x7c00  
   c          
   x /10i $pc  
   ```

   在lab1目录下执行`make debug`得到：

   ![1552808680159](C:\Users\fengs\AppData\Roaming\Typora\typora-user-images\1552808680159.png)

   可见在0x7c00处的断点设置成功

3. ### 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和bootblock.asm进行比较。

   首先修改Makefile文件在调用qemu时增加-d in_asm -D q.log 参数，然后执行make debug

   在/lab1/bin/q.log可以找到执行的汇编指令，与/lab1/boot/bootasm.asm中的代码基本相同，而与bootasm.S有所差别，主要在于两点：

   - 反汇编的代码中的指令不带指示长度的后缀，而bootasm.S的指令则有。比如，反汇编 的代码是`xor %eax, %eax`，而bootasm.S的代码为`xorw %ax, %ax`
   - 反汇编的代码中的通用寄存器是32位（带有e前缀），而bootasm.S的代码中的通用寄存器是16位（不带e前缀）。

   单步跟踪得到的部分代码如下

   ```x86
   ------
   
   IN: 
   0x00007c00:  cli    
   
   ------
   
   IN: 
   0x00007c01:  cld    
   
   ------
   
   IN: 
   0x00007c02:  xor    %ax,%ax
   ```

   

4. ### 自己找一个bootloader或内核中的代码位置，设置断点并进行测试。

   将端点设在0x7d10，tools/gitinit文件改为如下命令：

   ```
   file bin/kernel
   target remote :1234
   set architecture i8086
   b *0x7d10
   c
   x/i $pc
   set architecture i386
   define hook-stop
   x/i $pc
   end
   ```

   效果如图：

   ![1552815295600](C:\Users\fengs\AppData\Roaming\Typora\typora-user-images\1552815295600.png)

   

## 练习3：分析bootloader进入保护模式的过程。（要求在报告中写出分析）

### 1.关中断，将段寄存器置为0

```
start:
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    cld                                             # String operations increment

# Set up the important data segment registers (DS, ES, SS).

xorw %ax, %ax                                   # Segment number zero
movw %ax, %ds                                   # -> Data Segment
movw %ax, %es                                   # -> Extra Segment
movw %ax, %ss                                   # -> Stack Segment
```

### 2.开启A20

为了使 80286 和 8086/8088 在 real mode 下的行为一致，即：在 80286 下也产生 wraparound 现象。IBM 想出了古怪的方法：当 80286 运行在 real mode 时，将 A20 地址线（第 21 条 address bus）置为 0 ，这样使得 80286 在实模式下第 21 条 address line 无效，从而人为造成了 wraparound 现象。

为了使能所有地址位的寻址能力，需要打开A20地址线控制，即需要通过向键盘控制器8042发送一个命令来完成。

打开 A20 gate 的方法是给 keyboard controller 8042 发送 A20 gate enable 命令字，就是下面代码的 0xDF 命令。

具体步骤大致如下（参考bootasm.S）：
1. 等待8042 Input buffer为空；
2. 发送Write 8042 Output Port （P2）命令到8042 Input buffer；
3. 等待8042 Input buffer为空；
4. 将8042 Output Port（P2）得到字节的第2位置1，然后写入8042 Input buffer；

```
# Enable A20:
    #  For backwards compatibility with the earliest PCs, physical
    #  address line 20 is tied low, so that addresses higher than
    #  1MB wrap around to zero by default. This code undoes this.
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1

```

### 3.初始化全局描述符表，从实模式切换到保护模式

使用加载全局描述符表寄存器的函数lgdt加载GDT表，将CR0的保护允许位PE(Protedted Enable)置1，开启保护模式，然后跳转到处于32位代码块中的下一条指令

    # Switch from real to protected mode, using a bootstrap GDT
    # and segment translation that makes virtual addresses
    # identical to physical addresses, so that the
    # effective memory map does not change during the switch.
    lgdt gdtdesc
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
    
    # Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.
    ljmp $PROT_MODE_CSEG, $protcseg
### 4.重置其他段寄存器

```
.code32                                             # Assemble for 32-bit mode
protcseg:
    # Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment
```

### 5.设置栈帧指针，并调用c代码

    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
    call bootmain
    # If bootmain returns (it shouldn't), loop.
    spin:
        jmp spin
### 6.进入bootmain

保护模式完成，进入boot主方法

```
call bootmain
```

## 练习4：分析bootloader加载ELF格式的OS的过程。（要求在报告中写出分析）

通过阅读bootmain.c，了解bootloader如何加载ELF文件。通过分析源代码和通过qemu来运
行并调试bootloader&OS，

#### 1.1一些常量定义：

```c
#define SECTSIZE        512  // 扇区大小为512
#define ELFHDR          ((struct elfhdr *)0x10000)      // scratch space
```

#### 1.2 waitdisk函数：等待硬盘准备好

我们接下来需要的就是把硬盘中从第二个扇区开始的数据读入进来。0x1F0-0x1F7是内存中的特殊片段，我们可以用inb指令向这些位置写入数据用以向磁盘说明我们要读的数据。

```
static void
waitdisk(void) {
    while ((inb(0x1F7) & 0xC0) != 0x40)  // 等待磁盘准备好
        /* do nothing */;
}
```

#### 1.3readsect函数：读入一个扇区

dst是存储数据的位置，secno是扇区的编号。

```c
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);              // 向`0x1f2`端口写入希望读取的扇区个数；
    //向`0x1f3-0x1f6`端口写入希望读取的扇区号；
    outb(0x1F3, secno & 0xFF);   // 28位偏移量的第0-7位
    outb(0x1F4, (secno >> 8) & 0xFF);  // 28位偏移量的第8-15位
    outb(0x1F5, (secno >> 16) & 0xFF);  // 28位偏移量的第16-23位
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);  // 28位偏移量的24-27位；第28位置为0（磁盘0）；29-31位必须为1
    outb(0x1F7, 0x20);                      // 命令0x20：读扇区

    // 等待磁盘准备好
    waitdisk();

    // read a sector
    //读取一个扇区的内容到指定虚拟内存中。
    insl(0x1F0, dst, SECTSIZE / 4);  // 获得128个“长字”（4字节）大小的数据，即512字节
}
```

#### 1.4 readseg函数

从内核的offset处读count个字节到虚拟地址va中。复制的内容可能比count个字节多。

```c
static void
readseg(uintptr_t va, uint32_t count, uint32_t offset) {
    uintptr_t end_va = va + count;

    // 向下舍入到扇区边界
    va -= offset % SECTSIZE;

    // 从字节转换到扇区；kernel开始于扇区1
    uint32_t secno = (offset / SECTSIZE) + 1;

    // 如果这个函数太慢，则可以同时读多个扇区。
    // 我们会写得超出内存边界，但这并不重要：
    // 因为是以内存递增次序加载的。
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);
    }
}
```

#### 1.5 bootmain函数：bootloader的入口函数

在bootmain函数中，包含了加载OS的过程，见Step1-4：

```c
void
bootmain(void) {
    // Step1：读磁盘中的第一页
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // Step2：检查是否为ELF
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // Step3：加载描述表，并按照描述表读入ELF文件中的数据
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);// ph为program header的位置
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);// 虚拟地址的后8位，段的大小，段偏移量：读入对应的扇区
    }

    // Step4：根据ELF头部储存的入口信息，找到内核的入口。
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```

### 步骤分析

读取硬盘扇区的步骤为：

- 等待硬盘准备好
- 向内存中0x1F2-0x1F7区域中写入数据，说明需要读取的扇区的信息
- 等待硬盘准备好
- 从0x1F0处读入一个扇区的数据

加载kernel的步骤为：

​	获得了读取磁盘扇区的能力之后，Bootloader就可以读取ELF格式的操作系统文件了，具体的步骤如下：

- 将ELF文件头的8个扇区读取到内存的`0x10000`区域，作为ELF头的临时存储便于后续的访问，这部分区域不会被之后的拷贝操作所覆盖。
- 判断ELF文件是否合法（即检测文件头部magic值）。
- 从ELF头部读取Program Header表的偏移和长度。
- 遍历所有的Program Header：拷贝其中定义的数据到虚拟内存地址中去。拷贝以扇区为单位。
- 跳转到ELF头定义的起始运行地址开始执行。操作系统内核的启动时栈是基于Bootloader设立的运行栈的。

## 练习5：实现函数调用堆栈跟踪函数

对 kern/debug/kdebug.c：print_stackframe(）进行了补充：

```c
void print_stackframe(void) {
     /* LAB1 YOUR CODE : STEP 1 */
    //(1) call read_ebp() to get the value of ebp. the type is (uint32_t);
    uint32_t ebp = read_ebp();
    //(2) call read_eip() to get the value of eip. the type is (uint32_t);
    uint32_t eip = read_eip();
    //(3) from 0 .. STACKFRAME_DEPTH
    while(ebp){
        //    (3.1) printf value of ebp, eip
        cprintf("ebp = 0x%08x  eip = 0x%08x \n" , ebp , eip);
        //    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (uint32_t)ebp +2 [0..4]
        uint32_t arg1 = *((uint32_t *)ebp + 2);
        uint32_t arg2 = *((uint32_t *)ebp + 3);
        uint32_t arg3 = *((uint32_t *)ebp + 4);
        uint32_t arg4 = *((uint32_t *)ebp + 5);
        cprintf("arg1 = 0x%08x arg2 = 0x%08x arg3 = 0x%08x arg4 = 0x%08x \n" , arg1 , arg2 , arg3 , arg4);
        //    (3.3) cprintf("\n");
        //    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
        print_debuginfo(eip - 1);
        //    (3.5) popup a calling stackframe
        //           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
        //                   the calling funciton's ebp = ss:[ebp]
        eip = *((uint32_t *)ebp+1);
        ebp = *((uint32_t *)ebp);
    }
}
```

x86 中，栈向低地址增长，因此栈底在高地址，栈顶在低地址。

x86 中被调用函数的返回地址储存在调用函数栈帧的顶部，也就是最低的几个字节。对于当前（被调用）函数栈帧来说，就是栈帧底部以上的部分。调用函数的栈底信息则储存在当前函数栈帧的底部几个字节。因此在折叠栈的时候 `pop` 到 ebp 即可，再 `pop` 一个到 eip 就返回了（当然这一步是由 `ret` 指令实现的）。

x86 中的传参约定为，调用函数传入的参数应当从右至左压入栈中，因此参数地址可以直接由 ebp 推导出。

最后一行的内容是bootmain.c中的bootmain函数，也即第一个使用该堆栈的函数。bootloader设置的堆栈从0x7c00开始，使用“call bootmain”转入bootmain函数。 call指令压栈，所以bootmain中ebp为0x7bf8。

![1552923076753](C:\Users\fengs\AppData\Roaming\Typora\typora-user-images\1552923076753.png)

## 练习6：完善中断初始化和处理 （需要编程）

1. 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

  8个字节

  2、3字节是段选择子，0、1字节和6、7字节拼成位移， 两者联合便是中断处理程序的入口地址。

  

2. 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。

  

  ```c
  void
  idt_init(void) {
      extern uintptr_t __vectors[];
      int i = 0;
      for (; i < 256; ++i) {
          SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
      }
      SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
      lidt(&idt_pd);
  }
  ```

  

3. 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。

  在tarp_dispatch()中的case IRQ_OFFSET + IRQ_TIMER中加入以下代码

  ```c
          ticks++;
          if (ticks== TICK_NUM) {
              ticks= 0;
              print_ticks();
          }
  ```

## Challenge 1

扩展proj4,增加syscall功能，即增加一用户态函数（可执行一特定系统调用：获得时钟计数值），当内核初始完毕后，可从内核态返回到用户态的函数，而用户态的函数又通过系统调用得到内核态的服务（通过网络查询所需信息，可找老师咨询。如果完成，且有兴趣做代替考试的实验，可找老师商量）。需写出详细的设计和分析报告。完成出色的可获得适当加分。

从内核态切换到用户态的时候，ISR刚刚运行时系统还处在内核态，可以正常地对段寄存器进行修改；而相反过程在执行的时候，执行到`int`指令的时候或者执行到ISR的`movw %ax, %ds`一步时，就会触发一般保护异常（13号），导致无法正常执行ISR。所以，需要提前在IDT表中做出约定，这个约定包括两条：

- KERNEL_CS：在执行ISR的时候CS段需要为内核态；
- DPL_USER：本中断可以在用户态进行调用。

```c
void
idt_init(void) {
    extern uintptr_t __vectors[];
    int i = 0;
    for (; i < 256; ++i) {
        if (i == T_SYSCALL || i == T_SWITCH_TOK) {
			SETGATE(idt[i], 1, KERNEL_CS, __vectors[i], DPL_USER);
		} else {
			SETGATE(idt[i], 0, KERNEL_CS, __vectors[i], DPL_KERNEL);
		}
    }
    SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
    lidt(&idt_pd);
}
```

通过CS和DS等寄存器判断当前是否处于内核态或是用户态，切换的时候仅仅需要将这些寄存器的值进行相应的修改即可。
即需要在 `case T_SWITCH_TOU:` 和 `case T_SWITCH_TOK:` 两个 case 处添加修改段寄存器的代码即可：

```c
static void switch_to_user(struct trapframe *tf){
    if (tf->tf_cs != USER_CS) {
          tf->tf_cs = USER_CS;
          tf->tf_ds = USER_DS;
          tf->tf_es = USER_DS;
          tf->tf_ss = USER_DS;
          tf->tf_eflags |= FL_IOPL_MASK;
        }
}
static void switch_to_kernel(struct trapframe *tf){
    if (tf->tf_cs != KERNEL_CS) {
          tf->tf_cs = KERNEL_CS;
          tf->tf_ds = KERNEL_DS;
          tf->tf_es = KERNEL_DS;
          tf->tf_ss = KERNEL_DS;
          tf->tf_eflags &= ~FL_IOPL_MASK;
        }
}

    case T_SWITCH_TOU:
        switch_to_user(tf);
        break;
    case T_SWITCH_TOK:
        switch_to_kernel(tf);
```



而在中断处理例程中，原程序运行时的所有段寄存器均被压入栈中，处理完毕之后再弹出，因此考虑直接在中断处理例程中修改这些段寄存器的值从而达到用户态和内核态转换的目的。

在lab1_switch_to_user中，调用T_SWITCH_TOU中断。 注意从中断返回时，会多pop两位，并用这两位的值更新ss,sp，损坏堆栈。 所以要先把栈压两位，并在从中断返回后修复esp。

```c
static void
lab1_switch_to_user(void) {
    asm volatile (
    		"sub $0x8, %%esp \n"
    		"int %0 \n"
    		"movl %%ebp, %%esp \n"
    		:
    		:"i"(T_SWITCH_TOU)
    	);
}
```

在lab1_switch_to_kernel中，调用T_SWITCH_TOK中断。 注意从中断返回时，esp仍在TSS指示的堆栈中。所以要在从中断返回后修复esp。

```c
static void
lab1_switch_to_kernel(void) {
	asm volatile (
			"int %0 \n"
			"movl %%ebp, %%esp \n"
			:
			:"i"(T_SWITCH_TOK)
		);
}
```

扩展练习 Challenge 2（需要编程）

用键盘实现用户模式内核模式切换。具体目标是：“键盘输入3时切换到用户模式，键盘输入0
时切换到内核模式”。 基本思路是借鉴软中断(syscall功能)的代码，并且把trap.c中软中断处理
的设置语句拿过来。

在产生键盘中断时，对键盘的值进行判断，如果输入是‘0’或者‘3’就执行特权级转换的代码即可。

```c
case IRQ_OFFSET + IRQ_KBD:
        c = cons_getc();
        cprintf("kbd [%03d] %c\n", c, c);
        if (c == '0') {	
        	switch_to_kernel(tf);
        	print_trapframe(tf);
        }
        if (c == '3') {
        	switch_to_user(tf);
        	print_trapframe(tf);
        }
        break;
```

## 与参考答案的实现区别

- 练习1：我对各参数也进行了功能说明，但是答案更为细致，我主要是根据makefile较为深入浅出的讲解了镜像文件 ucore.img 是如何一步一步生成的，这点上比答案更加好理解一些。
- 练习2：断点测试上和答案基本一致，我比参考答案更加详细地介绍了单步跟踪的代码与源代码不同之处的原因
- 练习3：比答案更加细致的解释了A20开启过程背后的原因。
- 练习4：与参考答案相比，我补充了GDT表和保护模式下的段机制，并给出了uCore中GDT建立的细节分析。
- 练习5：代码的实现与标准答案基本相同。
- 练习6：基本一致
- challenge1：我将转换过程中对寄存器的修改写成函数形式，复用性更强；并且我对在idt_init中，将用户态调用SWITCH_TOK中断的权限打开这一操作的原因分析的更加细致。
- challenge2：调用的是我在上一个challenge中写的函数，更简洁。

## 本次实验各个练习对应OS原理的知识点

- 练习1：MBR扇区格式
- 练习2：x86的实模式与保护模式、内存布局、指令集
- 练习3：Bootloader的启动过程、GDT段机制访存和映射规则
- 练习4：ELF文件格式和硬盘访问
- 练习5：函数调用栈的结构
- 练习6：中断处理向量和中断描述符表的相关知识

## 实验中没有对应上的知识点

段机制的建立