# RotchOS-001: 框架

<div align="right"><font size="1" color="grey" align="right">Author: Rotch<br>Date: 2022-02-02</font></div>

### 关于 Multiboot

本操作系统使用 Multiboot 规范。

Multiboot 是一种引导操作系统的规范。**基本上，它指出了引导程序和操作系统之间的接口，这样任何符合规范的引导加载程序能够加载任何符合的操作系统。该规范没有指定引导加载程序应该如何工作，而是规定了它们必须如何与正在加载的操作系统一起使用该接口。**

### Multiboot 规范详解

一个 OS 映像可以是一个普通的操作系统使用的标准格式的 32 位可执行文件，不同之处是它可能被连接到一个非默认的载入地址以避开PC的 I/O 区域或者其它的保留区域。除了 OS 映像所使用的格式需要的头之外，OS 映像还必须额外包括一个 Multiboot 头。Multiboot 头必须完整的包含在 OS 映像的前 8192 字节内，并且必须是 `DWORD`（32位）对齐的。通常来说，它的位置越靠前越好，并且可以嵌入在 text 段的起始处，位于真正的可执行文件头之前。

Multiboot 分为 4 个部分：头分布、magic 域、地址域和图形域。

#### Multiboot 的头分布

Multiboot 头的分布必须如下表所示：

| 偏移量 | 类型   | 域名          | 必要性                |
| ------ | ------ | ------------- | --------------------- |
| 0      | uint32 | magic         | 必需                  |
| 4      | uint32 | flags         | 必需                  |
| 8      | uint32 | checksum      | 必需                  |
| 12     | uint32 | header_addr   | 如果 flags[16] 被置位 |
| 16     | uint32 | load_addr     | 如果 flags[16] 被置位 |
| 20     | uint32 | load_end_addr | 如果 flags[16] 被置位 |
| 24     | uint32 | bss_end_addr  | 如果 flags[16] 被置位 |
| 28     | uint32 | entry_addr    | 如果 flags[16] 被置位 |
| 32     | uint32 | mode_type     | 如果 flags[2] 被置位  |
| 36     | uint32 | width         | 如果 flags[2] 被置位  |
| 40     | uint32 | height        | 如果 flags[2] 被置位  |
| 44     | uint32 | depth         | 如果 flags[2] 被置位  |

Multiboot 的头需要三个必要变量：`magic`、`flags` 和 `checksum`。

其中，`magic` 必须等于 `0x1BADB002`，`checksum` 必须等于 `-(magic + flags)`。而 `flags` 指出 OS 映像需要引导程序提供或支持的特性：

> 其中 0-15 位指出需求：如果引导程序发现某些值被设置但出于某种原因不理解或不能不能满足相应的需求，它必须告知用户并宣告引导失败；16-31 位指出可选的特性：如果引导程序不能支持某些位，它可以简单的忽略它们并正常引导；所有 flags 字中尚未定义的位必须被置为 0。这样，flags 域既可以用于版本控制也可以用于简单的特性选择。

如果设置了 `flags` 字中的 0 位，所有的引导模块将按页（4KB）边界对齐。有些操作系统能够在启动时将包含引导模块的页直接映射到一个分页的地址空间，因此需要引导模块是页对齐的；

如果设置了 `flags` 字中的 1 位，则必须通过 Multiboot 信息结构（参见引导信息格式）的 `mem_*` 域包括可用内存的信息。如果引导程序能够传递内存分布（`mmap_*`域）并且它确实存在，则也包括它。我们设置第 0 位和第 1 位，则 `flags = (1 << 0) | (1 << 1) = 3`；

如果设置了 `flags` 字中的 2 位，有关视频模式表的信息必须对内核有效。

如果设置了 `flags` 字中的 16 位，则 Multiboot 头中偏移量 8-24 的域有效，引导程序应该使用它们而不是实际可执行头中的域来计算将 OS 映象载入到那里。如果内核映象为 `ELF` 格式则不必提供这样的信息，但是如果映象是 `a.out` 格式或者其他什么格式的话就必须提供这些信息。兼容的引导程序必须既能够载入 `ELF` 格式的映象也能载入将载入地址信息嵌入 Multiboot 头中的映象；它们也可以直接支持其他的可执行格式，例如一个 `a.out` 的特殊变体，但这不是必需的。

#### Multiboot 的地址域

所有由 flags 的第 16 位开启的地址域都是物理地址，它们的意义如下：

##### header_addr
包含对应于 Multiboot 头的开始处的地址——这也是 `magic` 值的物理地址。这个域用来同步 OS 映象偏移量和物理内存之间的映射。
##### load_addr
包含 `text` 段开始处的物理地址。（`load_addr` 必须小于等于 `header_addr`）
##### load_end_addr
包含 `data` 段结束处的物理地址。`(load_end_addr - load_addr)` 指出了引导程序要载入多少数据。这暗示了 `text` 和 `data` 段必须在 OS 映象中连续；现有的 `a.out` 可执行格式满足这个条件。如果这个域为 0，引导程序假定 `text` 和 `data` 段占据整个 OS 映象文件。
##### bss_end_addr
包含 `bss` 段结束处的物理地址。引导程序将这个区域初始化为 0，并保留这个区域以免将引导模块和其他的于查系统相关的数据放到这里。如果这个域为 0，引导程序假定没有 `bss` 段。
##### entry_addr
操作系统的入口点，引导程序最后将跳转到那里。

#### Multiboot 的图形域
所有的图形域都通过 `flags` 的第2位开启。它们指出了推荐的图形模式。注意，这只是 OS 映象推荐的模式。如果该模式存在，引导程序将设定它，如果用户不明确指出另一个模式的话。否则，如果可能的话，引导程序将转入一个相似的模式，他们的意义如下：

##### mode_type

如果为 0 就代表线性图形模式，如果为 1 代表标准 EGA 文本模式。所有其他值保留以备将来扩展。注意即使这个域为 0，引导程序也可能设置一个文本模式。

##### width

包含列数。在图形模式下它是象素数，在文本模式下它是字符数。0 代表 OS 映象对此没有要求。

##### height

包含行数。在图形模式下它是象素数，在文本模式下它是字符数。0 代表 OS 映象对此没有要求。

##### depth

在图形模式下，包含每个象素的位数，在文本模式下为 0。0 代表 OS 映象对此没有要求。

#### 机器状态

这里简单介绍 EAX、EBX 两个寄存器的机器状态。
当引导程序调用32位操作系统时，机器状态必须如下：

##### EAX

必须包含魔数 `0x2BADB002`；这个值指出操作系统是被一个符合 Multiboot 规范的引导程序载入的（这样就算是另一种引导程序也可以引导这个操作系统）。

##### EBX

必须包含由引导程序提供的 Multiboot 信息结构的物理地址

### 编写 boot.s

Multiboot 详解部分对于初学者来说可能比较难，因此我们简单使用该规范：

将以下代码保存为 `boot/boot.s`

```assembly
.set MGC, 0x1badb002
.set FLG, (1 << 0 | 1 << 1)
.set CKS, -(MGC + FLG)

.section .multiboot
    .long MGC
    .long FLG
    .long CKS

.section .text
.extern kernelMain
.global boot

boot:
    mov $kernel_stack, %esp
    push %eax
    push %ebx
    call kernelMain

.section .bss
.space 2 * 1024 * 1024  ; 2 MiB
kernel_stack:
```

**注意：汇编相关内容请自行学习，可参考文献推荐部分**

### 编写内核主函数

我们注意到第 18 行有一句 `call kernelMain`，但 `kernelMain` 却还没有定义，我们可以将以下代码保存为 `kernel.cpp`，写入以下内容：

```C++
extern "C" void kernelMain(const void* multiboot_structure, uint32_t argv)
{
    output("Welcome to RotchOS");
	for (;;)	// 让 CPU 不断执行循环操作，保持系统运行。
}
```

这样，我们就成功导入了 C++ 代码，系统也会从 `kernelMain` 开始执行。

以下是 `output(char *)` 的实现：

```C++
#define CalcPos(x, y)   ((y << 6) + (y << 4) + x)	// pos = y * 80 + x

static uint8_t x = 0, y = 0;
void ros_output_str(char* str)
{
    // 这里使用显卡的文字模式，内存从 0xb8000 处开始
    // 显存用于存储显示器需要显示的内容，是内存的一部分
    static uint16_t* VideoMemory = (uint16_t*)0xb8000;
    static uint32_t pos;

    // 屏幕大小为 80 * 25
    for (int i = 0; str[i] != '\0'; ++i)
    {
        switch(str[i])
        {
            // 处理换行
            case '\n':
                x = 0, ++y;
                break;
            // 其他字符正常输出
            default:
                pos = CalcPos(x, y),
                // 黑底白字显示字符
                VideoMemory[pos] = (VideoMemory[pos] & 0xFF00) | str[i],
                ++x;
                break;
        }
		
        // 处理换行
        if (x >= 80)	x = 0, ++y;
		// 屏幕写满则清屏
        // 注意：这个只是暂时的写法，后期会作修改
        if (y >= 25)
        {
            for (y = 0; y < 25; ++y)
                for (x = 0; x < 80; ++x)
                    pos = CalcPos(x, y),
                    VideoMemory[temp_val] = (VideoMemory[temp_val] & 0xFF00) | ' ';
            x = 0, y = 0;
        }
    }
}
```

附注：为方便将 C++ 的变量类型与内存大小相对应，我们进行简单的转换：

```C++
#ifndef _TYPES_H_
#define _TYPES_H_

typedef char                     int8_t;
typedef unsigned char           uint8_t;
typedef short                   int16_t;
typedef unsigned short         uint16_t;
typedef int                     int32_t;
typedef unsigned int           uint32_t;
typedef long long int           int64_t;
typedef unsigned long long int uint64_t;

typedef unsigned int             size_t;
typedef uint8_t                    BYTE;
typedef uint16_t                   WORD;
typedef uint32_t                  DWORD;

#endif /* _TYPES_H_ */
```

（保存为 `include/types.h`）

### 编译

将以下内容保存为 `boot/linker.ld`

```assembly
ENTRY(boot)
OUTPUT_FORMAT(elf32-i386)
OUTPUT_ARCH(i386:i386)

SECTIONS
{

  .text :
  {
    *(.multiboot)
    *(.text*)
    *(.rodata)
  }

  .data :
  {
    KEEP(*( .init_array ));
    KEEP(*(SORT_BY_INIT_PRIORITY( .init_array.* )));

    *(.data)
  }

  .bss :
  {
    *(.bss)
  }

  /DISCARD/ :
  {
    *(.fini_array*)
    *(.comment)
  }
}
```

接下来我们书写 `Makefile`：

```makefile
GCCPARAMS = -m32 -Iinclude -fno-use-cxa-atexit -nostdlib -fno-builtin -fno-rtti -fno-exceptions -fno-leading-underscore -Wno-write-strings
ASPARAMS = --32
LDPARAMS = -melf_i386

objects = \
	obj/boot.o \
	obj/kernel.o

obj/%.o: %.cpp
	mkdir -p $(@D)
	gcc $(GCCPARAMS) -o $@ -c $<

obj/%.o: boot/%.s
	mkdir -p $(@D)
	as $(ASPARAMS) -o $@ $<

Kernel.bin: boot/linker.ld $(objects)
	ld $(LDPARAMS) -T $< -o $@ $(objects)

run: RotchOS.iso clean

RotchOS.iso: Kernel.bin
	rm -rf $@
	mkdir iso
	mkdir iso/boot
	mkdir iso/boot/grub
	cp $< iso/boot/
	echo 'set timeout=0'                         > iso/boot/grub/grub.cfg
	echo 'set default=0'                        >> iso/boot/grub/grub.cfg
	echo ''                                     >> iso/boot/grub/grub.cfg
	echo 'menuentry "Rotch Operating System" {' >> iso/boot/grub/grub.cfg
	echo '  multiboot /boot/Kernel.bin'         >> iso/boot/grub/grub.cfg
	echo '  boot'                               >> iso/boot/grub/grub.cfg
	echo '}'                                    >> iso/boot/grub/grub.cfg
	grub-mkrescue --output=$@ iso
	rm -rf iso

clean:
	rm -rf obj Kernel.bin *.o

```

这里对于 `linkerscript` 和 `makefile` 不作讲解，请自行学习。

这里使用 GRUB 启动方式，我们暂且可以不关注 `grub.cfg` 的编写。

这样，我们的操作系统框架就做好了，使用命令行输入 `make run` 即可生成 `RotchOS.iso`。该镜像使用虚拟机即可运行。

### 文献推荐

本书综合参考了一些操作系统相关书籍，推荐如下：

1. 《30 天自制操作系统》（川合秀实【日】）
2. 《x86汇编语言：从实模式到保护模式》（李忠、王晓波、余洁）
3. 《Linux 内核完全注释》（赵炯）

本篇文章 Multiboot 部分参考博客 [Multiboot 规范（uframer）](https://blog.csdn.net/uframer/article/details/373900)。

参考网站：https://wiki.osdev.org/

