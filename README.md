# apb

## 4 rCore教学操作系统设计与实现

教学操作系统实验有了硬件平台支持后，本章根据所搭建的硬件平台属性设计了教学操作系统实验。其中教学操作系统实验设计分为三层，分别为引导程序、操作系统内核、用户程序的设计与实现，通过上述三层结构构成完整的软件系统。

![](https://raw.githubusercontent.com/liuxuesong123/pic/master/WX20200226-140716%402x.png)

​ 图x 操作系统实验设计结构图

参考实现各模块关系如图x所示，引导程序设计分为ROM与SDRAM两段引导，包括设备初始化、解压缩、内核与用户程序加载等功能并在4.1节中实现。操作系统内核模块实现中断处理、系统调用、用户程序解析等功能。用户程序设计为如何让操作系统加载解析ELF用户程序，跳转至用户空间执行用户程序。通过操作系统实验的三层设计实现可帮助学生理解操作系统设计方法，深化对操作系统运行过程的理解。引导程序、操作系统内核模块与用户程序设计具体参考实现如下。

### 4.1 引导程序的设计与实现

引导程序的功能为初始化硬件系统、解析并加载操作系统内核至指定地址，并跳转至操作系统运行。受FPGA片上资源的限制，本实验针对picorv32 CPU设计了特定的引导程序。其具体设计思路为：首先，引导程序初始化硬件设备，提供操作系统内核运行的基本环境。其次，设计实现硬件平台初始化工作，对片上ROM以及片外SDRAM进行初始化操作。接下来，设计实现系统调用程序并移植到引导程序中，为串口驱动程序提供调用接口，实现操作系统实验平台与上位机的交互，便于开发者对引导程序启动过程的检测。然后，由于受到片上存储空间以及操作系统启动方式的限制，裁剪解压缩函数库并移植到引导程序中，在占用极少片上存储空间的前提下，为引导程序提供解压缩功能，并将压缩后的操作系统镜像移植到解压缩函数库内，保证了解压缩函数运行的唯一性，以便提升操作系统启动的安全性。接下来，引导程序调用解压缩函数库，解压缩函数将嵌入自身的操作系统压缩镜像解压至外部存储空间SDRAM指定地址。最后，由于进入操作系统启动流程后不再回到引导程序阶段，设计采用绝对跳转的方式，跳转至SDRAM指定地址完成剩余引导工作后直接进入操作系统入口地址，进入操作系统内核启动流程。

#### 4.1.1 引导程序具体功能与适应性改动

根据4.1节引导程序设计思路的安排，本文对引导程序的具体实现作出了如下修改：

为了简化操作系统内核代码的加载流程并减少硬件的使用资源，在获得操作系统镜像前还需对操作系统内核文件进行处理。其具体实现为将操作系统内核转化为数据段与代码段等不同程序段直接加载至引导程序指定位置，形成新的包含引导程序与操作系统内核代码的操作系统镜像，避免因解析操作系统elf文件而耗费更多硬件资源。

为了节省硬件平台片上存储空间，在引导程序中加入经过裁剪的解压缩模块，将操作系统镜像进行gzip压缩处理后，移植到经过裁剪的解压缩模块中。其具体实现为将加载了操作系统内核代码段，数据段，以及用户程序的引导程序编译链接为二进制文件，将该二进制文件进行gzip格式压缩，得到操作系统压缩镜像，通过文件转换器将该压缩镜像转换为数组。经过压缩的操作系统镜像大小可降至20KB以下，相较于操作系统直接从rom启动节约一半的存储空间。

为了方便测试操作系统运行的正确性，在引导程序中设计实现并移植了系统调用程序以便为串口驱动等程序提供调用接口，串口驱动程序的功能是为教学操作系统实验平台提供读写功能，既可从教学操作系统发送运行数据至上位机，也可从上位机接收运行命令并作出相应反馈。通过上述功能为实现教学操作系统与上位机的交互提供调试手段。

为了保证教学操作系统能够在外接的SDRAM存储空间上正确运行，在将操作系统压缩镜像转换的数组嵌入解压缩程序后，将其作为解压缩函数的输入数组，分别指定解压缩函数输入输出数组的内容及长度，通过引导程序完成硬件初始化功能后，通过汇编指令跳转至解压缩算法入口处，对包含操作系统内核代码段，数据段，用户程序并在sdram启动的RustOS镜像进行解压。

为了实现引导程序至操作系统的跳转，在解压完成后，通过设计绝对跳转的方式，跳转至存在于SDRAM的规定地址的教学操作系统起始地址处，进入教学操作系统RustOS的启动流程。在此需要注意的是，由于硬件平台的picorv32 CPU未能实现MMU，因此上述过程中所涉及到的地址空间均由本文在合理安排的前提下计算所得，因此绝对跳转的地址是由人工指定的，其优势在于在地址明确的前提下，可明确各段程序的起始与终止地址并实时监控引导程序与操作系统各个模块运行数据的正确性。

**运行流程设计**

在本文中，引导程序作为CPU执行的第一段代码，其起始地址为CPU启动地址，即0x0000\_0000处，根据硬件系统中CPU的配置，中断跳转地址为0x10，即引导程序在0x10处实现中断处理。

本文直接使用无条件跳转指令跳过中断处理程序入口，并转至初始化代码。初始化代码需将32个通用寄存器置零，其次，加载程序数据段、bss段以初始化内存，其中，bss数据段数值置零。

接下来，因需要通过串口实现操作系统实验平台与上位机的交互，所以需要设置串口分频系数。

随后初始化引导程序堆栈寄存器、全局寄存器与线程寄存器。

接下来引导程序打印“BOOT”表明初始化完毕。

接下来引导程序调用自定义指令，开启中断，通过ecall指令触发异常以检验中断是否成功开启，随后根据需求设置时钟计数器与led灯。

在完成硬件系统的初始化后，跳转至解压缩程序入口，进行操作系统镜像的解压缩工作。其被解压的引导程序除去完成中断处理、硬件初始化、串口分频系数设置等功能后，通过“.incbin”伪指令加载操作系统代码段至标号text\_payload\_start处，同时设置引导程序代码段地址偏移量为0x1000。

操作系统代码段加载完成后，同样通过伪指令“.incbin”将用户可执行程序加载至引导程序elf\_payload\_start标号处，并设置地址偏移量为0x9000。

引导程序数据段预留32个字大小空间作为中断处理程序保存寄存器现场的堆栈，预留128个字大小空间作为中断处理函数堆栈。

至此，引导程序运行完毕，由于操作系统内核已经内置在引导程序中，因此，引导程序运行完毕后，操作系统已经开始在sdram中执行。

**跳转指令设计**

在RustOS启动过程中，经历了两次跳转

（1）由引导程序到解压缩函数的跳转

RISC-V架构下的无条件跳转指令：j、jal、jr、jalr，其用法如下

| **指令** | **解析** | **示例** |
| :--- | :--- | :--- |
| j | 把 pc 设置为当前值加上符号位扩展的 offset，等同于 jal x0, offset。 | j offset |
| jal | J类指令，立即数+pc为跳转目标，rd存放pc+4（返回地址），   可用于进行子程序调用。 | jal rd, offset |
| jr | pc = x\[rs1\]，把 pc 设置为   x\[rs1\]，等同于 jalr x0, 0\(rs1\)。 | jr rs1 |
| jalr | I类指令，rs+立即数为跳转目标，rd存放pc+4（返回地址），   可用于子程序返回指令。 | jalr rd, offset\(rs1\) |

由于我们需要从引导程序跳转到解压缩函数入口处，该命令是在rom中完成，执行跳转后无需返回，所以我们在引导程序完成中断处理、硬件初始化、串口分频系数设置等功能后，执行如下指令：

```text
j decompress
```

decompress 为解压缩函数，解压缩函数定义如下：

```text
int decompress(unsigned char *out, unsigned int *dlen, const void *in,unsigned int slen)
```

其解压缩函数的参数分别为解压缩后数组、解压缩后数组大小、压缩镜像数组、压缩镜像数组大小。经实验可知解压缩后数组位于bss段，所以可以通过对链接脚本进行修改，来改变解压缩后镜像的物理起始地址。

（2）由解压缩函数到RustOS起始地址的跳转

在解压缩工作完成后，通过如下指令进行绝对地址跳转：

```text
((void()(void))address)();
```

其中，\(void\(_\)\(void\)是和int_ 相类似的数据类型，int _是指向int型的指针，而\(void\(_  \)\(void\)是一个指向函数的指针，且这个函数无返回值，无参数。之后通过\(  _\(void\(_  \)\(void\)\)做强制类型转换，接着把address强制转化为一个函数指针，即\(  _\(void\(_ \)\(void\)\)address，最后一步就是调用这个函数，跳转到指定的绝对地址去执行。address为我们想要跳转到的物理地址，此地址为在SDRAM启动的RustOS的入口地址。

#### 4.1.2 sbi设计

SBI，即Supervisor Binary Interface。为实现对上层系统提供调用接口的功能，引导程序通过ecall软中断的形式提供系统调用接口。由于中断处理函数没有预留参数作为系统调用号，设计将自定义系统调用函数的参数传递规则。系统调用函数的参数传递规则如表所示。

| **寄存器** | **功能** |
| :--- | :--- |
| a0~a3 | 系统调用函数参数 |
| a7 | 系统调用号 |
| a0 | 系统调用函数返回值 |

中断处理函数通过读取regs数组中偏移量为10~13（a0~a3）的值得到调用函数传递的三个参数，通过读取regs数组中偏移量为17（a7）的值得到系统调用类型。在中断处理函数中，定义系统调用型号的枚举类型如下。

```text
enum   sbi_call_t { SBI_CONSOLE_PUTCHAR = 1, SBI_CONSOLE_GETCHAR = 2, } ;
```

本系统仅设计实现两种系统调用。

（1）对于SBI\_CONSOLE\_PUTCHAR系统调用，即向console打印一个字符，其通过将第一个参数a0传入的的值传递给由输入输出驱动提供print\_char\(\)函数实现功能，返回值为0。

（2）对于SBI\_CONSOLE\_GETCHAR系统调用，即向console打印一个字符，其通过调用输入输出驱动提供的getchar\(\)函数，对字符进行接收。

#### 4.1.3 解压缩模块设计

为解决片上ROM存储空间不足导致内核镜像不能加载的问题，引导程序中的解压缩模块对内核映像进行压缩处理，为操作系统的启动提供支持。实验将操作系统内核镜像压缩为gzip格式文件后，转换为数组嵌入引导程序。引导程序调用解压缩库tinf\[11\]将内核压缩镜像解压至SDRAM，并通过绝对跳转的方式跳转至内核入口执行操作系统程序。此外，由于直接调用tinf解压缩库会占用太多的存储空间，因此实验需要对tinf解压缩库进行适应性裁剪，最终达到节省一半ROM存储空间的效果。接下来对裁减与移植的思路作出详细阐述。

#### 4.1.4 解压缩模块tinf的裁剪与移植

若对解压缩模块tinf进行裁剪，首先需要明确解压缩模块tinf 的组成，包括解压缩模块支持的功能以及所引用的各个函数库等。未经裁减的tinf解压缩库函数支持zlib，gzip和zip格式的解压缩，其中deflate算法规范来自DEFLATE压缩数据格式规范版本1.3，zlib数据格式来自ZLIB压缩数据格式规范版本3.3，gzip数据格式来自GZIP文件格式规范版本4.3。tinf解压缩数据库已经假定为其提供了有效的压缩数据，并且有足够的空间对解压缩数据进行存放。如下图所示为tinf解压缩库的文件构成

![](https://raw.githubusercontent.com/liuxuesong123/pic/master/%E5%9B%BE%E7%89%87%201.png)

其中tgunzip文件夹内容为针对gzip压缩文件的测试文件，test文件夹内容包括文件如下图

![](https://raw.githubusercontent.com/liuxuesong123/pic/master/%E5%9B%BE%E7%89%87%202.png)

图X文件所提供的功能为检测待解压缩文件或数组是否符合标准压缩文件格式，并通过test\_tinf.c文件对解压缩库功能进行检测。

Src文件夹包含文件如图x所示，该文件夹文件功能是为tinf解压缩库针对gzip、zlib等压缩文件提供解压缩功能源代码。

![](https://raw.githubusercontent.com/liuxuesong123/pic/master/%E5%9B%BE%E7%89%87%203.png)

​ 上述两个文件中的文件为本文主要使用的文件，并在上述文件的基础上选择适当格式的解压缩功能，并对为使用到的不同格式的解压缩模块部分进行裁剪，以便尽量少的占用存储空间。

​ 考虑到教学操作系统的设计实现环境为Linux实验环境，从综合角度来看Gzip相较于tinf解压缩库中支持的其他几种解压缩格式，压缩与解压速度较快，压缩率较高，处理该格式的压缩文件方式较为方便，并且大部分Linux系统支持gzip命令，因此在完成教学操作系统设计后可轻易获得gzip格式的教学操作系统压缩镜像。因此本文选用了gzip压缩合适作为教学操作系统压缩与解压的使用格式。

在确定使用gzip压缩格式后，本文对tinf解压缩库进行裁剪。由上文可知，首先tinf解压缩库支持zlib，gzip和zip格式的解压缩，因此需要去除tinf对除gzip外的zlib等格式的解压缩功能支持。首先从功能的角度判断，在对上述文件编译前去除tinfzlib.c文件，该文件的功能是为tinf 提供zlib格式文件的解压缩。其次，由于gzip解压缩功能建立在deflate算法功能至上，因此需要保留tinfgzip.c与tinflate.c文件。此外，保留crc32.c文件作为检测手段保证解压缩数据的正确性。最后tinf.h与greatest.h作为头文件保留下来，至此从文件的角度对tinf解压缩库完成了初步裁剪。

接下来还需要对头文件tinf.h内的具体代码作出修改，如图x所示。在头文件中删除对zlib格式解压缩功能的支持。

![img](https://github.com/oscourse-tsinghua/thesis-lxsong2020/blob/master/%E7%B4%A0%E6%9D%90/%E5%9B%BE%E7%89%87%204.png>)

除去删除类似图x所示tinf对其他格式解压缩功能的支持，还需要自行完成makefile文件的编写以测试在裁剪后解压缩模块是否能够正确运行。因此本文设计如下makefile文件，具体实现如下。

```text
CFLAGS=                        -D_POSIX_C_SOURCE=200112L -D_BSD_SOURCE
CFLAGS+=          -g -std=c99 -pedantic
CFLAGS+=          -DTEST_PROG
CFLAGS+=          -DMINI_GZ_DEBUG
all: mini_gzip
mini_gzip: decompress.c crc32.c tinfgzip.c tinflate.c makefile
   gcc $(CFLAGS) -Wall -c decompress.c
   gcc $(CFLAGS) -Wall -c crc32.c
   gcc $(CFLAGS) -Wall -c tinfgzip.c
   gcc $(CFLAGS) -Wall -c tinflate.c
   gcc $(CFLAGS) -Wall decompress.o crc32.c tinfgzip.c tinflate.c -o test
clean:
   rm -rf  *.o test
```

为了完成完整的检测过程，本文另行编写了测试文件，来完成测试，测试文件具体实现如下：

```text
#include "tinf.h"
#include <stdlib.h>
#include <stdio.h>
#include <time.h>
#include "greatest.h"
#ifndef ARRAY_SIZE
#  define ARRAY_SIZE(arr) (sizeof(arr) / sizeof((arr)[0]))
#endif
   static const unsigned char data[] = {“压缩后的文件数组”}；
const unsigned char a[] = {“压缩前的文件数组” };
int main(void)
{
        tinf_init();
        unsigned char out[49000] ;
   unsigned int dlen =  ARRAY_SIZE(out);
        tinf_gzip_uncompress(out, &dlen, data, ARRAY_SIZE(data));
int i = 0;
    int flag = 0;
    for (i=0;i<9672;i++)
    {
       if (a[i] != out[i])
       {
         flag = 1;
         break;
       }
    }
    if (flag == 1)
    printf("neq\n");
    else
    printf("eq\n"); 
}
```

本文对测试文件的设计思路是将压缩后的文件转换为数组后添加至 static const unsigned char data\[\] = {“压缩后的文件数组”}；处。将压缩前的文件转换为数组后添加至const unsigned char a\[\] = {“压缩前的文件数组” };处。在完成解压缩工作后，将压缩后的文件数组解压至指定位置后与压缩前的文件数组逐一对比，若两数组数据完全相同，则解压成功，同时在终端打印“eq”。

![](https://raw.githubusercontent.com/liuxuesong123/pic/master/WX20191126-093143%402x.png)

通过分析本文编写的测试文件可发现，本文裁剪后的tinf所处理的压缩文件并不是传统意义上的以gzip文件形式存在的文件，而是被转换为数组形式。目的是直观快速的完成解压缩工作，同时也保证了解压缩过程的稳定性。因此在将tinf模块移植至引导程序前，本文已将教学操作系统完成gzip格式压缩并通过文件转换器转换为了数组形式添加至tinf解压缩模块中。在引导程序中通过跳转指令跳转至解压缩模块后完成解压工作。本文解压流程的具体实现如下。

```text
int decompress(unsigned char *out, unsigned int *dlen, const void *in,unsigned int slen)
//int decompress()
{
        print_str("decompress OS......\r\n");      
        tinf_init();
        tinf_gzip_uncompress(out, &dlen, in, slen);
        print_str("decompress finished!\r\n");
        (*(void(*)(void))0x010002a8)();
    return 0;
}
```

调用解压缩程序完成解压工作后，向串口打印"decompress finished!\r\n"提示用户解压缩工作已完成。通过 \(_\(void\(_\)\(void\)\)0x010002a8\)\(\);跳转至地址0x010002a8处进行操作系统的运行，跳转的具体地址信息上文已作出解释，在此不再重复。

此外，教学操作系统所实现的功能都需要根据存储空间与解压缩函数的性能作出适应性调整，接下来对教学操作系统的设计与实现作出详细阐述。

### 4.2 操作系统内核的设计与实现

#### 4.2.1 rCore教学操作系统设计基础

操作系统内核的设计依赖于硬件平台资源，STEP-CYC10开发板上可用的资源包括 48KB片上存储资源、UART、GPIO、数码管等。其中，操作系统代码压缩后存储至ROM，解压缩至SDRAM后使系统能够被CPU 正常加载执行。SoC中的串口设备保证了操作系统基本的输入输出功能，便于用户与实验平台上的操作系统进行交互。最后，在SDRAM开辟空间保证操作系统在运行时能够加载数据和设置堆栈等程序运行所必须的资源支持

#### 4.2.2 最小化内核实验

从本质上讲，操作系统是一个独立式可执行程序。这决定了操作系统的编写过程与一般直接在系统中运行的可执行程序不同，其最大特点是无法使用依赖于某特定平台的函数库。此外，由于编译器没有符合硬件平台的目标结构，因此其编译也必须以特殊方式进行。

**新建系统工程**

使用cargo new 创建一个新的项目 xy\_os。命令如下

$ cargo new xy\_os

创建完成后进入os项目文件夹，并尝试构建、运行项目，指令如下：

$ cargo run

此时我们可以看到终端打印出“Hello，world！”，表明该程序已经可以正常运行，但即使是上述简单的打印功能，也离不开操作系统（ubuntu开发环境）的支持，我们的目的是写一个新的操作系统，所以理论上讲不能依赖于任何已有操作系统。且该项目是针对x86平台Linux操作系统构建的可执行项目，因此需更改编译目标架构，以适配RISC-V指令集。

**更改编译的目标架构**

cargo 在编译项目时，可以附加目标参数 --target  设置项目的目标平台。平台包括硬件和软件支持，事实上， 目标三元组包含：cpu 架构、供应商、操作系统和ABI。

在安装rust时，默认编译后的可执行文件要在本平台执行，可使用rustc --version --verbose来查看rust的默认目标三元组：

```text

```

可以看到Rust以LLVM作为编译器后端，其默认的目标三元组，CPU架构为x86\_64，供应商为unknown，操作系统为linux，ABI为gun。

通过执行$ rustup target list并截取部分结果如下：

```text
mips-unknown-linux-gnu
riscv32imac-unknown-none-elf
riscv32imc-unknown-none-elf
riscv64gc-unknown-none-elf
riscv64imac-unknown-none-elf
```

由输出结果可知Rust编译器支持riscv32架构，由于官方提供的目标三元组都不适用于编写的新操作系统，所以需要使用json文件定义适用于新操作系统的目标三元组，在编译时通过--target json文件名指定\[16\]。

新建文件xy\_os.json，文件内容如下：

```text
{ "llvm-target": "riscv32",
"data-layout": "e-m:e-p:32:32-i64:64-n32-S128",
"target-endian": "little",
"target-pointer-width": "32",
"target-c-int-width": "32",
"os": "none",
"arch": "riscv32",
"cpu": "generic-rv32",
"features": "+m",
"max-atomic-width": "32",
"linker": "rust-lld",
"linker-flavor": "ld.lld",
"pre-link-args": {
"ld.lld": ["-Tsrc/boot/linker.ld"] },
"executables": true,
"panic-strategy": "abort",
"relocation-model": "static",
"eliminate-frame-pointer": false
```

可以看到里面描述了架构为riscv，端序为小端序等信息，同时指定自定义链接脚本对操作系统进行链接。

**移除标准库依赖**

通过上文叙述可知，项目默认链接rust标准库std，并默认使用标准库中的println！宏进行格式化输出，它依赖于操作系统因此需要将标准库禁用，方式如下。

```text

```

对大多数语言来说，他们都使用了运行时系统，这导致main并不是他们执行的第一个函数。rust语言规定，链接了标准库的rust程序会首先跳转到C runtime library 中的 crt0\(C runtime zero\) 进入C runtime设置 C 程序运行所需要的环境\(比如：创建堆栈，设置寄存器参数等\)。然后 C runtime 会跳转到 rust runtime 的 入口点\(entry point\) 进入rust runtime继续设置rust运行环境，而这个入口点就是被start语义项标记的。rust runtime 结束之后才会调用 main 进入主程序。C runtime 和rust runtime都需要标准库支持，我们的程序无法访问。所以需要加入如下命令，表示不再链接标准库，不再使用rust默认的函数入口点\[16\]。

```text
#![no_main]
```

当rust程序发生异常时需要有相应的函数对其进行处理，标准库中对应的函数为panic，当移除标准库依赖后，panic无法被编译器使用，为保证程序正常运行，需实现与panic同名函数，实现方式如下。

```text
use core::panic::PanicInfo;
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
   loop {}
}
```

程序在panic后本应该结束，我们暂时先设置panic\_handler进入死循环，因此panic\_handler不会结束，用！类型的返回值表明该函数不会返回。

在上述过程中，我们用到了rust语言中不依赖于平台的core库，该库主要包含了基础的rust类型，如 Result、Option等。panic 函数使用core库中的错误信息类型 PanicInfo作为参数。

标准rust程序在进行panic时，会进行堆栈展开，所谓堆栈展开，是指当程序出现不可恢复的错误时，我们需要沿着调用栈一层一层回溯上去回收每个caller中定义的局部变量避免造成内存溢出，以达到释放内存的目的。此时会用到标准库语义项‘eh\_personality’\[16\]，其中eh时 exception handling 的简写，它是一个标记某函数用来实现堆栈展开处理功能的语义项。为简单起见，我们不考虑内存溢出，设置当程序panic时不进行任何清理工作，直接退出程序即可。这样堆栈展开函数就不会被调用，其实现方式为修改 Carao.toml 文件，具体如下所示。

```text
[profile.dev]
panic = "abort"
[profile.release]
panic = "abort"
```

**堆栈设置**

由引导程序启动方式可知，引导程序未设置操作系统堆栈，所以在跳转至操作系统的main函数运行前，需要先设置堆栈。可使用汇编指令完成初始化堆栈功能。rust语言可通过 global\_asm！宏使用内嵌汇编指令，通过 include\_srt！将特定文件作为字符串包含到目标文件中\[16\]。指令如下。

```text
#![feature(global_asm)]
global_asm!(include_str!("entry.asm"));
```

通过 \#!\[feature\(global\_asm\)\] 开启rust对汇编指令的支持。堆栈设置方式如下。

```text
.section .text
  .global _start
_start: 
  lui sp, %hi(bootstacktop)
  addi sp, sp, %lo(bootstacktop)
call main
  .section .data
  .align 4 
  .global bootstack
bootstack:
  .space 2048
  .global bootstacktop
bootstacktop:
```

在data数据段中从标号bootstack到标号bootstacktop处预留2KB空间作为操作系统的堆栈空间，由于堆栈地址自高向低增长，所以将bootstacktop赋值给sp寄存器。完成初始化工作后，通过call指令跳转到rust程序main函数执行下一步工作。

**main函数实现**

在main.rs中添加如下指令。

```text
#[no_mangle]
pub extern "C" fn main() { }
```

\#\[no\_mangle\]的功能使编译器禁用 name mangling ，确保编译器生成一个名为 main 的函数。由于从entry.asm

跳转到main函数遵循 C 调用规则，所以使用extern "C" 使编译器遵循 C calling convention，pub则保证了该函数

名可以被 entry.asm访问。

rust编译器在链接时使用rust lld，在禁用标准库后，出现如下错误：

```text
rust lld: error: undefined symbol: abort
```

解决上述错误，需手动实现 abort 函数作为标号提供给 rust lld，方式如下。

```text
#[no_mangle]
pub extern fn abort() {
  panic!("abort!");
}
```

#### 4.2.3 中断处理实验

**中断处理**

根据硬件系统配置可知，中断处理程序入口地址为0x10，由于启用了q0~q3，所以可使用q2~q3作为scratch来存储x1、x2寄存器的值，通过picorv32\_setq\_insn自定义指令将x1和x2的值设置到q2和q3之中。

接下来中断处理程序将irq\_regs赋给x1寄存器，irq\_regs为在SDRAM中分配给中断处理程序保存寄存器的地址，用以存储引发中断时各通用寄存器的值，并将中断返回地址存储在偏移量为0的地址处，其余寄存器按照下标存储在n\*4偏移地址处。

然后中断处理程序将irq\_stack赋给sp寄存器，irq\_stack为在SDRAM中分配给中断处理程序的堆栈地址。

中断处理程序将irq\_regs和q1寄存器的值作为参数，并通过a0，a1寄存器传入，调用中断处理函数irq解析中断类型并做出相应处理。

完成上述操作从中断处理函数irq返回后，中断处理程序将恢复中断现场，重新设置时钟计数器。并调用picorv32\_retirq\_insn自定义指令返回原程序继续执行。

因为CPU进入异常模式时自动屏蔽中断，即不允许嵌套中断，故无需额外加入代码开关中断。

**中断处理函数**

中断处理函数通过C语言编写，其声明形式在头文件firmware.h中，并在文件irq.c中实现，为避免跨平台导致数据长度冲突，在头文件中引入stdint.h，在编写程序时可直接使用C99标准库定义的数据类型。

中断处理函数声明如下：

```text
uint32_t * irq(uint32_t *regs, uint32_t irqs)
```

中断函数需要两个传递参数，第一个参数为指向存储中断现场通用寄存器的内存指针，第二个参数为中断掩码类型，以上两个参数分别通过寄存器a0、a1传入。中断处理函数的返回值为指向存储通用寄存器的内存指针。

**判断中断类型**

系统预设的中断信号类型有3类，分别对应下标为0、1、2的中断掩码位，其中，中断掩码位为0，则为时钟中断，中断掩码位为1，则为异常指令ebreak，ecall与非法指令，中断掩码位为03，则表示访存地址不对齐异常，中断处理函数需根据传入的中断掩码给出中断类型。

**时钟中断**

对于时钟中断，系统的处理方式为通过全局静态变量存储时钟出发次数，每次触发时钟中断，便向串口打印\[TIMER-IRQ\]+时钟中断次数。

**异常指令中断**

对于异常指令中断，需进一步判断触发中断的原因。存储中断现场通用寄存器的内存空间regs的0地址偏移处存储的值为中断处理程序返回地址，根据异常实现原理，对于ECALL、EBREAK一类触发软中断的异常指令，其返回地址为触发软中断指令的下一条指令地址。由此可计算处异常中断地址，方式如下。

```text
uint32_t pc = (reg[0] & 1) ? regs[0] – 3 : regs[0] – 4;
```

pc值为异常指令地址，以字为单位对齐。通过访问内存将该地址处指令读取并做进一步判断。方式如下。

```text
uint32_t instr = *(uint32_t *)pc；
```

根据RISC-V指令编码手册，比较 instr 的值可判断异常指令类型。如下表所示。

| **指令类型** | **编码** |
| :--- | :--- |
| ebreak | 0x0010000073 |
| ecall | 0x00000073 |
| 非法指令 | 其余情况 |

当运行遇到ebreak或非法指令时，系统将打印所有寄存器的值，作为调试信息对软件进行调试。

**访存地址不对齐中断**

针对方寸地址不对齐中断，系统处理方式为向串口打印“Bus Error”，并停机等待重启。

#### 4.2.4 系统调用实验

由上文可知，引导程序向操作系统提供基本输入输出系统调用，操作系统通过汇编指令 ecall，同时传入相应参数进行调用。为方便使用，本文创建相应的库对系统调用进行封装，操作系统通过导入外部库的方式使用系统调用功能。

首先执行如下命令创建库项目bbl\[17\]。

```text
cargo new --lib bbl
```

并修改lib.rs内容。

```text
#![no_std]
#![feature(asm)]
pub mod sbi;
```

上述指令分别表示不适用标准库，开启内敛汇编并声明子模块sbi。sbi模块通过sbi.rs实现，功能为对系统调用函数进行封装，实现方式如下。

```text
pub fn console_putchar(ch: usize) {
  sbi_call(SBI_CONSOLE_PUTCHAR, ch, 0, 0);
}
pub fn console_getchar() -> usize {
  sbi_call(SBI_CONSOLE_GETCHAR, 0, 0, 0)
}
#[inline(always)]
fn sbi_call(which: usize, arg0: usize, arg1: usize, arg2: usize) -> usize {
  let ret;
  unsafe {
      asm!("ecall"
          : "={x10}" (ret)
          : "{x10}" (arg0), "{x11}" (arg1), "{x12}" (arg2), "{x17}" (which)
          : "memory"
          : "volatile");
  }
  ret
}
const SBI_CONSOLE_PUTCHAR: usize = 1;
const SBI_CONSOLE_GETCHAR: usize = 2;
```

将bbl库复制到项目根目录中，并更改 Cargo.toml 。

```text
[dependencies]
bbl = {path = "bbl/"}
```

将该库加入项目依赖中。至此，操作系统可调用该库当中的函数完成数据输入输出操作。

#### 4.2.5 串口驱动实验

**串口驱动设计**

操作系统通过串口与上位机进行交互，通过串口向上位机发送数据。上位机接收后，通过串口调试工具进行显示。串口功能直接通过Quartus RS-232 IP核实现。驱动程序需访问和使用的寄存器包括rxdata、txdata、status寄存器，其偏移量分别为0、1、2，以字为单位。由上文设置可知，串口设备的基址为0x0200\_0000，故分别对三个寄存器的基址的进行宏定义如下。

```text
#define INPORT ((volatile uint32_t)0x02000000) //uart_rx
#define OUTPORT ((volatile uint32_t)0x02000004)   //uart_tx
#define uart_status ((volatile uint32_t)0x02000008)
```

寄存器以字为单位，所以使用uint32\_t类型指针指向内存映射的UART设备寄存器。通过volatile关键字避免编译器对使用该变量值的地方做过多优化。

（1）print\_char\(\)函数的实现

printf\_char\(\)函数的功能为向txdata寄存器发送数据，输入参数为需发送的字符，返回值为0。在发送数据之前，调用trdy函数判断当前串口的状态，若为可发送状态，则向txdata寄存器发送数据。此外，驱动程序还在print\_char\(\)函数的基础上，实现打印字符串的函数print\_str\(\)，打印十进制数字的函数print\_dec\(\)以及打印十六进制的函数print\_hex\(\)。

（2）getchar\(\)函数的实现

getchar\(\)函数的功能为从rxdata寄存器读取数据，无输入参数，返回值为读取的寄存器数据并将其转换为char类型。在读取数据之前，调用rrdy函数判断当前串口的状态，若为可读取状态，则从rxdata寄存器读取数据。

#### 4.2.6 格式化输出实验

格式化输出具体实现过程（附加代码）

**创建IO模块**

在一个文件内实现过多的功能会导致文件过于冗长，不利于代码的阅读与维护，所以本文在main.rs的同级目录下创建一个新的文件 io.rs 用于管理io。首先在文件 io.rs 中实现用于输入输出的 putchar 和 puts 函数，实现方式如下\[16\]。

```text
use bbl::sbi;
pub fn putchar(ch: char) {
  sbi::console_putchar(ch as u8 as usize);
}
pub fn puts(s: &str) {
  for ch in s.chars() {
      putchar(ch);
  }
}
```

从上述代码函数名可以看出，两个函数的功能分别是输出单个字符和输出字符串。

在main.rs中引入 io 库：

```text
pub mod io；
```

至此完成 io 模块的创建与使用。

**println！宏的实现**

想要实现println！宏，print！必不可少，首先来实现print！，方式如下

```text
#[macro_export]
macro_rules! print {
  ($($arg:tt)*) => {
      $crate::io::_print(format_args!($($arg)*));
  }
}
```

\#\[macro\_export\]宏使外部库也可以使用这个宏。format\_args! 宏可以将print（…）内的部分转换为 fmt::Arguments类型，用以后续打印。在这里我们还用到了一个未实现的函数\_print。其实现方法如下：

首先引入 fmt::Write 特征（trait）\[16\]，创建一个新的类 StdOut。实现 write\_str（）方法，write\_str\(\)方法的功能为，将String Slice作为参数传入，然后将该String Slice输出，返回Result类型。仅当成功写入整个String Slice时,此方法返回Ok\(\(\)\), 在写入所有数据或发生错误之前, 此方法不会返回。

```text
struct StdOut;
impl fmt::Write for StdOut {
  fn write_str(&mut self, s: &str) -> fmt::Result {
      puts(s);
      Ok(())
  }
}
```

接下来对\_print进行实现：

```text
pub fn _print(args: fmt::Arguments) {
  StdOut.write_fmt(args).unwrap();
}
```

rust core库中带有用于格式化和打印字符串的库，该库中的trait—write可将消息格式化为数据流。根据trait的特性，仅需为Write trait 实现 write\_str（）方法即可使用该trait中的其余方法。由于我们已经实现了write\_str，接下来核心库会自动实现格式化输出方法 write\_fmt。

最后我们来完成println！宏的实现：

```text
#[macro_export]
macro_rules! println {
  () => (print!("\n"));
  ($($arg:tt)) => (print!("{}\n", format_args!($($arg))));
}
```

println！宏的实现包含两条匹配规则，匹配完成后的语句均在print！宏的基础上加入“\n”输出。然后通过\#\[macro\_export\]将print！宏导出至根目录。从本质上看，IO 模块仅需要实现 \_print函数就可以实现println！宏，此时IO 模块如下。

```text
use bbl::sbi;
use core::fmt::{self, Write};
pub fn putchar(ch: char) {
  sbi::console_putchar(ch as u8 as usize);
}
pub fn puts(s: &str) {
  for ch in s.chars() {
      putchar(ch);
  }
}
struct Stdout;
impl fmt::Write for Stdout {
  fn write_str(&mut self, s: &str) -> fmt::Result {
      puts(s);
      Ok(())
  }
}
pub fn _print(args: fmt::Arguments) {
  Stdout.write_fmt(args).unwrap();
}
#[macro_export]
macro_rules! print {
  ($($arg:tt)*) => {
      $crate::io::_print(format_args!($($arg)*));
  }
}
#[macro_export]
macro_rules! println {
  () => {
      print!("\n");
  };
  ($($arg:tt)*) => {
      print!("{}\n", format_args!($($arg)*));
  }
}
```

### 4.3 用户程序解析实验

elf 文件解析实现如下。

```text
use core::mem::transmute;
use core::slice;
//use core::ptr;
use xmas_elf::{
    header,
    ElfFile,
};
#[no_mangle]
pub extern "C" fn elf_interpreter() {
    let _elf_payload_start = 0x9000;
    let _elf_payload_end = 0xae3c;
    let kernel_size = _elf_payload_end as usize - _elf_payload_start as usize;
    let kernel = unsafe { slice::from_raw_parts(_elf_payload_start as *const u8, kernel_size) };
    let kernel_elf = ElfFile::new(kernel).unwrap();
    header::sanity_check(&kernel_elf).unwrap();
    for program_header in kernel_elf.program_iter() {
         println!("{:?}", program_header);
    }
    let entry = kernel_elf.header.pt2.entry_point() as u32;
    let kernel_main: extern "C" fn( ) = unsafe { transmute(entry) };
    println!("kernel_main: {:#?}", kernel_main);
//   copy_kernel(_elf_payload_start as usize, &segments);
    kernel_main( );
}
```

解析elf文件需要提供elf文件地址，标号\_elf\_payload\_start、\_elf\_payload\_end分别代表了elf文件的起始地址与终止地址。该elf用户程序通过伪指令.incbin嵌入到引导程序中，并可以根据需求指定起始地址。接下来通过from\_raw\_parts（）函数截取从起始地址到种植地址的整个elf用户程序。使用xmas\_elf::ElfFile结构体将kernel变量类型强制转换为xmas\_elf库中定义的ElfFile结构体类型。此时，依据xmas\_elf库中的函数对ELF文件进行解析。其中，header::sanity\_check（）为检查elf文件的头文件，确保elf文件的正确性。接下来通过program\_iter\(\)函数遍历elf文件各段信息，再通过函数header.pt2.entry\_point\(\)解析出elf文件用户程序入口点。程序入口点地址将存入变量entry中，若想要跳转至elf程序入口点执行，需要将变量entry转换为函数形式。在本工程中通过函数transmute（）将变量类型转换为extern "C" fn\( \)函数类型。最后调用转换成函数的kernel\_main，跳转至elf程序入口点执行下一步操作。

### 4.4 程序链接与加载设计

picorv32 CPU 并未实现 MMU，即内存管理单元，并且内存资源受到限制，操作系统无法实现内存管理功能。本文所有程序使用地址均为人工设置的物理地址，其各代码与数据地址已在链接脚本中指定，从本质上看，对引导程序的地址空间分配即为对全局的地址空间进行分配管理。

根据引导程序的设计，操作系统与用户程序最终都要嵌入到引导程序的指定标号处。在嵌入操作系统与用户程序前，需要通过链接脚本对引导程序进行链接。在链接脚本中使用 MEMORY 命令设置内存的起始地址与大小，并指定内存类型。ROM为只读类型，SDRAM为可读可写类型。指定ROM存储可执行代码段与只读数据，SDRAM存储数据段与bss段。由于引导程序没有加载程序用来加载数据段。所以在链接脚本中需要使用 AT 命令指定数据段的加载地址，并通过汇编指令将数据段从ROM复制到SDRAM。

对于在SDRAM启动的引导程序，在完成寄存器和设备初始化以及串口分频系数、中断设置后，跳转到操作系统可执行代码段起始地址标号text\_payload\_start处运行。当引导程序编译为elf可执行文件后。可通过反汇编命令查看标号text\_payload\_start的地址，同理数据段起始地址标号data\_payload\_start的地址，为方便起见，可通过伪指令 .org 指定标号起始地址偏移量。在本文中，指定text\_payload\_start偏移量为0x1000，data\_payload\_start偏移量为0x300，此时可确定操作系统可执行代码段的起始地址为0x01001000，操作系统数据段的起始地址为0x01100300。

在操作系统项目中新建链接脚本文件linker.ld。在该脚本中，操作系统在SDRAM中执行，所以分别为操作系统代码段、数据段、bss段分别指定到SDRAM中的有效地址即可，通过上文可知，操作系统各段起始地址。

由于存储空间的限制，本文只需提取rcore的可执行代码段与数据段嵌入到引导程序。同时省去了引导程序解析elf程序功能，内核链接地址更容易计算，也更容易节省内存空间。

与操作系统连接到引导程序的方法类似，用户程序直接通过ELF文件嵌入至引导程序标号为\_elf\_payload\_start地址处。同样可通过反汇编命令查看标号的具体地址。

由于\_elf\_payload\_start标号地址与用户程序可执行代码段起始地址不同，所以不能将标号地址作为用户程序的链接地址。若要确定用户程序可执行代码段的起始地址，首先需要通过反汇编命令定位用户程序用来初始化堆栈的代码。然后与用户程序源程序对比，确定用户程序可执行代码段起始地址。

若根据上述地址修改用户程序连接脚本，则用户程序新生成的elf文件的结构将会发生变化，导致将用户程序嵌入到引导程序相同地址时，可执行代码段的偏移量也会随之发生变化。当操作系统解析出用户程序入口点时，跳转地址仍为先前的可执行代码段地址。使得操作系统无法正常进入用户程序。

为解决上述问题，本文同时固定标号\_elf\_payload\_start地址与用户程序可执行代码段起始地址，其用户程序可执行代码段起始地址在用户程序链接脚本中指定。

假设用户程序链接脚本中代码段起始地址为addr\_ld，引导程序中用户可执行代码段起始地址为addr\_load，偏移量为offset，引导程序标号地址为addr\_elf。为使标号 \_elf\_payload\_start 地址与用户程序中可执行代码段入口地址相同，可通过如下公式计算标号 \_elf\_payload\_start 地址具体地址。

offset=addr\_ld-addr\_load

此时保证用户程序不变，修改标号\_elf\_payload\_start地址

addr\_elf=addr\_elf+offset

最后使用伪指令 .org 指定\_elf\_payload\_start标号地址为addr\_elf。

在将操作系统与用户程序链接完成后，本实验软件系统各部分地址分布如图x所示。

![](https://raw.githubusercontent.com/liuxuesong123/pic/master/WX20200226-140758%402x.png)

### 4.5 本章小结

作为本文的主要工作，本章首先实现了引导程序的设计，由于受到硬件存储空间的限制，在引导程序中添加了解压缩模块，并详细介绍了解压缩模块的实现流程与启用方法。在完成引导程序的基础上，实现了操作系统的设计，其中逐次实现了最小化内核、系统调用、格式化输出、用户程序解析的功能。最后依据硬件存储设备提供的数据以及CPU功能的限制完成了操作系统各个模块的地址空间布局，并完成了与硬件平台的链接。实现了完成的教学操作系统设计。

