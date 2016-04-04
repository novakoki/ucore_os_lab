ucore Lab1
===

练习一
---

1. 
  Q: ucore.img 是如何一步步生成的？
  
  A: 让make输出执行的命令
  
    ```sh
      make V= | less
    ```
  
  从输出结果可以看到分为以下几个步骤：
    1. 编译kernel 
    
    ```sh
      + cc kern/*/*.c
      + cc kern/*/*.S
    ```    
    
    2. 链接生成kernel
    
    ```sh
      + ld bin/kernel
    ```
    
    3. 编译bootblock
    
    ```sh
      + cc boot/bootasm.S
      + cc boot/bootmain.c
    ```
    
    4. 编译sign工具
    
    ```sh
      + cc tools/sign.c
    ```
    
    5. 链接生成bootblock
    
    ```sh
      + ld bin/bootblock
    ```
    
    6. 最后生成ucore.img
    
    ```sh
      dd if=... of=bin/ucore.img ...
    ```
  
  Makefile中涉及到的主要命令及相关参数
  1. cc(假设没有用clang)
  
    ```sh
      gcc -I<dir> -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -c *.c -o *.o
    ```
    
    - -I 指定头文件寻找目录
    - -fno-builtin 不接受不是两个下划线开头的内建函数（这里不太明白）
    - -Wall 打开gcc的所有警告
    - -ggdb gdb调试
    - -m32 32位环境
    - -gstabs stabs格式调试信息
    - -nostdinc 不使用标准库（没有系统，自然也没有标准库）
    - -fno-stack-protector 不生成用于检测缓冲区溢出的代码
    - -c 源文件
    - -o 输出文件
    
  2. ld
  
    ```sh
      ld -m elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel obj/kern/*/*.o
      ld -m elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o 
    ```
  
    - -m 模拟指定的链接器，这里是32位(elf_i386)
    - -nostdlib 不使用标准库
    - -T 指定使用的脚本
    - -o 输出文件
    - -N 指定读取/写入文本和数据段
    - -e 使用指定的符号作为程序的初始执行点
    - -Ttext 使用指定的地址作为文本段的起始点（大概相当于 ORG 0x7C00？）
    
    这里应该还有一步用sign工具处理并生成bootblock的步骤，不知道为什么make没有输出相关信息。
    大致看了下sign.c的代码，应该就是填充整个文件到512个字节，并且最后两个字节为`0x55`,`0xAA`。
    
  3. dd
  
    ```sh
      # 生成一个10000个块的文件并用0填充
      dd if=/dev/zero of=bin/ucore.img count=10000
      # 第一个扇区用bootblock填入
      dd if=bin/bootblock of=bin/ucore.img conv=notrunk
      # 后续扇区填入kernel
      dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunk 
    ```
  
2. 
  Q: 符合规范的硬盘主引导扇区的特征是什么? 
  
  A: 总共512字节，最后两个字节是`0x55`,`0xAA`

*****
  
练习二
---

1. 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行
  
  这里课本上的内容和附录之间有点出入，课本练习二描述中查看当前运行代码的gdb命令应该是错了。
  第一条指令从地址`0xfffffff0`取，在实模式下即为`CS=0xf,IP=0xfff0`。
  
  1. 首先修改tools/gdbinit的内容如下
    
    ```
      define  hook-stop
      x/i     $cs*16+$pc 
      # x为examine的缩写，即查看内存内容
      # 课本中只写了$pc，而在实模式下指令的地址应为$cs*16+$pc
      end
      set architecture i8086
      target remote :1234
    ```
  
  2. 然后开始调试程序
    
    ```sh
      make debug
      # 前两条指令如下
      The target architecture is assumed to be i8086
      0xffff0:     ljmp   $0xf000,$0xe05b
      0x0000fff0 in ?? ()
      (gdb) si
      0xfe05b:     cmpl   $0x0,%cs:0x6574
      0x0000e05b in ?? ()
      (gdb) 
    ```

2. 在初始化位置0x7c00设置实地址断点,测试断点正常
  
  修改`tools/gdbinit`
  
  ```
    define  hook-stop
    x/i     $cs*16+$pc
    end
    set architecture i8086
    target remote :1234
    b *0x7c00
    continue  
  ```

   调试后输出如下

  ```sh
    Breakpoint 1 at 0x7c00
    => 0x7c00:      cli    

    Breakpoint 1, 0x00007c00 in ?? ()
    (gdb) 
  ```
  
3. 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和	bootblock.asm进行比较
  
  反汇编代码
  
  ```sh
    Breakpoint 1 at 0x7c00
    => 0x7c00:      cli    

    Breakpoint 1, 0x00007c00 in ?? ()
    (gdb) si
    => 0x7c01:      cld    
    0x00007c01 in ?? ()
    (gdb) si
    => 0x7c02:      xor    %ax,%ax
    0x00007c02 in ?? ()
    (gdb) si
    => 0x7c04:      mov    %ax,%ds
    0x00007c04 in ?? ()
    (gdb) si
    => 0x7c06:      mov    %ax,%es
    0x00007c06 in ?? ()
    (gdb) si
    => 0x7c08:      mov    %ax,%ss
    0x00007c08 in ?? ()
    (gdb) 
  ```
  
  bootasm.S
  
  ```asm
    cli                                             # Disable interrupts
    cld                                             # String operations increment

    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment
  ```
  
  bootblock.asm
  
  ```asm
    cli                                             # Disable interrupts
    7c00:	fa                   	cli    
    cld                                             # String operations increment
    7c01:	fc                   	cld    

    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   # Segment number zero
    7c02:	31 c0                	xor    %eax,%eax
    movw %ax, %ds                                   # -> Data Segment
    7c04:	8e d8                	mov    %eax,%ds
    movw %ax, %es                                   # -> Extra Segment
    7c06:	8e c0                	mov    %eax,%es
    movw %ax, %ss                                   # -> Stack Segment
    7c08:	8e d0                	mov    %eax,%ss
  ```
  
4. 自己找一个bootloader或内核中的代码位置,设置断点并进行测试

  这里我选择内核中的kene_init函数作为断点
  
  `tools/gdbinit`
  
  ```
    file bin/kernel
    target remote :1234
    break kern_init
    continue
  ```
  
  gdb显示
  
  ```sh
    Breakpoint 1, kern_init () at kern/init/init.c:17
    (gdb) 
  ```
  
  单步调试若干步后的QEMU输出
  
  ```
    (THU.CST) os is loading ...
    Special kernel symbols:
      entry  0x00100000 (phys)
      etext  0x00103209 (phys)
      edata  0x0010da16 (phys)
      end    0x0010ed20 (phys)
    Kernel executable memory footprint: 60KB
  ```

***
  
练习三
---

分析bootloader 进入保护模式的过程

1. 关中断，将各个段寄存器置0
2. 使能A20，这样才能访问1M以上的内存空间，这里主要做一个向8042写数据的操作
3. 设置GDT
4. 设置`CR0`寄存器最后一位为1
5. 做一个长跳转，到32位模式的代码处继续运行
      
***

练习四
---
1. 
  Q: bootloader如何读取硬盘扇区的？

  A: 
  
  `readsect`函数读取第secno个扇区到dst指向的内存
  
  `readseg`使用`readsect`函数读取count个扇区到va指向的内存
2. 
  Q: bootloader是如何加载ELF格式的OS？

  A: 
  ```c
    void bootmain(void) {
        // 读取ELF的头部
        readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
    
        // 判断是否合法
        if (ELFHDR->e_magic != ELF_MAGIC) {
            goto bad;
        }
    
        struct proghdr *ph, *eph;
    
        // ph指向ELF头部的描述表
        ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
        eph = ph + ELFHDR->e_phnum;
    
        // 按照描述表将ELF文件中数据载入内存
        for (; ph < eph; ph ++) {
            readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
        }

        // 内核入口
        ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
    
        bad:
            outw(0x8A00, 0x8A00);
            outw(0x8A00, 0x8E00);
            while (1);
    }
  ``` 
  
***

练习五
---
实现函数调用堆栈跟踪函数并解释最后一行各个数值的含义

最后一行如下

```
  <unknow>: -- 0x00007d67 --
```

根据`bootblock.asm`的内容

```asm
  // call the entry point from the ELF header
  // note: does not return
  ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
  7d5c:	a1 18 00 01 00       	mov    0x10018,%eax
  7d61:	25 ff ff ff 00       	and    $0xffffff,%eax
  7d66:	ff d0                	call   *%eax
```

那么`bootmain.c`中对应的就是

```c
  // call the entry point from the ELF header
  // note: does not return
  ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
```

也就是kernel的入口

***

练习六
---
Q: 中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

A: 中断向量表一个表项占用8个字节。
其中2-3字节是段选择子，0-1字节为偏移量的前16位，6-7字节为偏移量的后十六位。
 