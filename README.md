## 一、环境配置
STM32开发环境
## 二、获取官方固件包源文件
##### 为了获取最干净、标准的 HAL 库与 CMSIS 文件，我们需要下载 ST 官方的 STM32CubeF4 仓库。

你可以在电脑上任选一种方式下载：

- 方式 A（Git 命令行推荐，速度快且省去不需要的历史版本）：
    在工作目录打开终端，执行浅克隆：

    Bash
    ```
    git clone --depth 1 https://github.com/STMicroelectronics/STM32CubeF4.git
    ```
    (注：如果 GitHub 访问慢，也可以去 Gitee 镜像仓库搜索 STM32CubeF4 下载)

- 方式 B（浏览器直接下载 ZIP 包）：
前往 GitHub 的 STMicroelectronics/STM32CubeF4 仓库主页，点击绿色的 Code -> Download ZIP，然后解压到本地。
## 三、精简并抽取核心依赖
##### 如果把整个编译过程比作盖房子，我们需要四样东西：内核标准（图纸规范）、外设寄存器定义（房间门牌号）、启动汇编（地基施工队）以及硬件驱动库（现成的水暖电模块）。
1. 内核标准（图纸规范）
    ```
    stm32f405_minimal/
    ├── Core/
    │   ├── Inc/
    │   └── Src/
    └── Drivers/
        ├── CMSIS/
        │   ├── Device/
        │   │   └── ST/
        │   │       └── STM32F4xx/
        │   │           └── Include/
        │   └── Include/
        └── STM32F4xx_HAL_Driver/
            ├── Inc/
            └── Src/
    ```
   - Core 目录：存放我们自己编写的用户业务代码（如 main.c、中断回调），以及芯片特有的系统初始化文件。
   - Drivers 目录：存放官方提供的、不可随意篡改的底层驱动与硬件架构规范。这符合嵌入式领域通用的BSP/驱动与业务解耦思想。
   - CMSIS 是什么：它的全称是 Cortex Microcontroller Software Interface Standard（微控制器软件接口标准），由 ARM 官方制定。
      - 这里面最重要的文件是 core_cm4.h。
      - STM32F405 是基于 ARM Cortex-M4 内核的芯片。内核级别的硬件控制（例如 NVIC 中断控制器的使能、SysTick 定时器寄存器操作、MPU 内存保护单元、FPU 单精度浮点指令集）是由 ARM 统一设计的，与 ST 公司无关。编译器必须依赖这些头文件，才能理解 __enable_irq()、NVIC_SetPriority() 等底层内嵌汇编指令。
    > 具体操作：
    > 1. 打开解压后的固件包路径：
    >STM32CubeF4/Drivers/CMSIS/Include/
    > 2. 将这个文件夹内的所有 .h 头文件，全部复制到工程的：
    >Drivers/CMSIS/Include/ 目录下。

2. 外设寄存器定义（房间门牌号）
   - 外设寄存器映射 (==stm32f405xx.h==)：虽然内核是 ARM 设计的，但外设（GPIO、USART、TIM、SPI 等）是 ST 公司设计的。ST 为STM32F405 的每个寄存器分配了绝对物理地址（例如 GPIOA 的基地址是 0x40020000）。这个头文件通过结构体指针将内存地址映射为 C 语言对象，没有它，代码就无法识别诸如 GPIOA->MODER 这样的写法。
   - 时钟基础配置 (==system_stm32f4xx.c==)：芯片上电时默认运行在慢速的内部 RC 振荡器（HSI 16MHz）上。该文件定义了系统全局变量 SystemCoreClock，并提供了 SystemInit() 函数。该函数主要完成两项任务：
     1. 打开 Cortex-M4 的 CP10 和 CP11 协处理器控制寄存器，开启硬件浮点单元（FPU）。
     2. 确保中断向量表基地址偏移（VTOR）指向正确的 Flash 起始位置。

    > 具体操作：
    > 1. 提取芯片寄存器定义头文件：
    > 打开固件包路径：STM32CubeF4/Drivers/CMSIS/Device/ST/STM32F4xx/Include/
    > 复制以下 3 个文件到工程的 Drivers/CMSIS/Device/ST/STM32F4xx/Include/：
    >     - stm32f4xx.h
    >     - stm32f405xx.h
    >     - system_stm32f4xx.h
    > 2. 提取时钟初始化源文件：
    > 打开固件包路径：STM32CubeF4/Drivers/CMSIS/Device/ST/STM32F4xx/Source/Templates/
    > 复制 system_stm32f4xx.c 到工程的 Core/Src/ 目录下。
3. 启动汇编（地基施工队）
   - GNU 启动汇编 (==startup_stm32f405xx.s==)：微控制器上电时不能直接从 main() 函数开始跑。它需要一段汇编代码完成硬件准备：
     1. 定义中断向量表（放在内存起始位置 0x08000000，首个字为堆栈指针顶端 _estack，第二个字为复位入口 Reset_Handler）。
     2. 初始化 C 运行环境：把 Flash 里的常量数据复制到 RAM（.data 段），并把 RAM 里未初始化的全局变量清零（.bss 段）。
     3. 调用 SystemInit()，最后跳转到 main()。
    > 具体操作：
    > ###### 因为选择的工具链是 ARM GNU Toolchain (GCC)，所以必须选择 gcc/ 目录下的 startup_stm32f405xx.s。
    > 1. 提取启动汇编文件：打开固件包路径：
    >STM32CubeF4/Drivers/CMSIS/Device/ST/STM32F4xx/Source/Templates/gcc/
    > 2. 复制 startup_stm32f405xx.s 到工程的 Core/Src/ 目录下。

4. 硬件驱动库（现成的水暖电模块）
    - HAL 库之间存在较多依赖，但下述 6 个模块是 STM32 运行的“最小生命维持集”。先只保留这几个模块，能避免因为缺少其他外设宏而引发一连串难以排查的编译报错。后续需要串口或定时器时，再随时向里面追加 stm32f4xx_hal_uart.c 等文件即可。
    > 具体操作：
    > 1. 复制头文件：
    > 打开路径：STM32CubeF4/Drivers/STM32F4xx_HAL_Driver/Inc/
    > 复制以下头文件到工程的 Drivers/STM32F4xx_HAL_Driver/Inc/：
    >    - stm32f4xx_hal.h（HAL 核心总头文件）
    >    - stm32f4xx_hal_def.h（HAL 状态码与结构体定义）
    >    - stm32f4xx_hal_cortex.h（SysTick 定时器与中断封装）
    >    - stm32f4xx_hal_rcc.h、stm32f4xx_hal_rcc_ex.h（时钟复位控制）
    >    - stm32f4xx_hal_gpio.h、stm32f4xx_hal_gpio_ex.h（通用输入输出口，用于点灯）
    >    - stm32f4xx_hal_pwr.h、stm32f4xx_hal_pwr_ex.h（电源管理与超频调压）
    > 2. 复制对应的 C 源文件：
    > 打开路径：STM32CubeF4/Drivers/STM32F4xx_HAL_Driver/Src/
    > 复制以下 .c 文件到工程的 Drivers/STM32F4xx_HAL_Driver/Src/：
    >    - stm32f4xx_hal.c（初始化 HAL 计时滴答与内部机制）
    >    - stm32f4xx_hal_cortex.c
    >    - stm32f4xx_hal_rcc.c、stm32f4xx_hal_rcc_ex.c
    >    - stm32f4xx_hal_gpio.c
    >    - stm32f4xx_hal_pwr.c、stm32f4xx_hal_pwr_ex.c
## 四、编译器与链接脚本
1. 编译器是什么？
   - 编译器（Compiler）是一个翻译工具。微控制器（MCU）只认二进制机器指令（即 0 和 1 组成的机器码），不懂人类写的 C/C++ 或汇编代码。在工具包中，arm-gnu-toolchain 就是这一套工具的集合：
     - aarch64-none-elf-gcc：负责把 C 语言代码编译成汇编指令，再组装成包含机器码的目标文件（.o 文件）。
       - aarch64（目标架构 Architecture）
       编译器生成的是 ARM 64 位指令集（ARMv8-A、ARMv9-A 等）的机器码。它支持 64 位通用寄存器（X0~X30）、64 位地址寻址空间以及 A64 指令集。典型硬件如 Cortex-A53、Cortex-A72、树莓派 4B/5 以及各类高性能 64 位 SoC。
       - none（目标操作系统 Vendor / OS）
       目标平台处于无常规操作系统的裸机状态。代码直接运行在底层硬件上，没有运行标准的桌面或服务器 Linux 发行版。
       - elf（输出可执行文件格式）
       它的全称是 Executable and Linkable Format（可执行与可链接格式）。
       编译产物为通用的纯 ELF 二进制结构，包含段信息、符号表与重定位信息，用于在无特定 OS 约束的环境下烧录与加载。
       - gcc（核心编译器工具）
       GNU C/C++ 编译器套件。
      - arm-none-eabi-gcc
         - arm（目标架构 Architecture）
         编译器生成的目标机器指令属于 ARM 架构体系。它支持 Cortex-M 系列（如 STM32 采用的 M0/M3/M4/M7）以及 Cortex-R、旧版 ARM7/ARM9 等内核。
         - none（目标操作系统 Vendor / OS）
         目标平台没有安装常规操作系统（即裸机 Bare-Metal）。
         与之对比的是 arm-linux-gnueabihf-gcc，后者的中间字段是 linux，表示目标设备运行完整的 Linux 操作系统。
         - eabi（应用程序二进制接口 ABI）
         它的全称是 Embedded Application Binary Interface（嵌入式应用二进制接口）。
         它规定了底层硬件执行层面的核心规范，包括：
         函数调用时参数如何通过寄存器（如 R0~R3）或堆栈传递。
         数据类型在内存中的对齐规则（例如 32 位整型按 4 字节对齐）。
         浮点数计算的寄存器交互规范。
         - gcc（核心编译器工具）
           它的全称是 GNU Compiler Collection（GNU 编译器套件），表明它是基于开源 GCC 架构构建的 C/C++ 语言前端与优化生成器。
       - arm-none-eabi-as : 负责把启动文件（.s 文件）直接翻译成目标文件。
            > 可以看出启动文件不一定要汇编文件。
    - 编译器的局限性（单兵作战）：
         - 编译器在翻译时是逐个文件翻译的。
         - 当编译器在翻译 main.c 这一页时，它看到你写了一句“调用 LED_Init() 函数”，但这个函数写在 gpio.c 里面。
         - 这时编译器并不去翻 gpio.c。它只是把 main.c 翻译好，并在调用处留下一张便利贴：“这里需要调用一个叫 LED_Init 的函数，具体位置我先空着，等链接器来填”。
         - 编译器翻译完后，会给每个 .c 文件各生成一个 .o 文件（目标文件，相当于一页页翻译好的草稿）。
2. 链接器是什么？
    - 各个独立的 .o 文件无法直接烧录到芯片里跑，必须有一道工序把它们合并在一起，这个工具叫链接器（Linker，在 GCC 里是 arm-none-eabi-ld）。
    - 链接器做两件事：
       1.  符号解析：你在 main.c 里调用了 HAL_Init()，链接器负责从 stm32f4xx_hal.o 里找到该函数的实现，并把调用跳转指令绑定过去。
       2.  重定位与空间分配：把所有的指令、常量、全局变量分配到芯片的真实物理地址上。
3. 链接脚本是什么？
     - 链接脚本（Linker Script，即 .ld 文件）就是交予链接器的物理空间施工蓝图。
     - 链接器对你的硬件一无所知。它不知道你用的是 STM32F405 还是 STM32F103，更不知道你的芯片有几兆 Flash、多大的 RAM。如果不给它规则，它不知道该把代码塞到哪。
     - 链接脚本主要规定了两大核心要素：
         1. 硬件内存边界宣告(MEMORY)
             STM32F405VGT6 的硬件设计已经固定了物理存储区域：
             ```
             MEMORY
             {
                 RAM    (xrw) : ORIGIN = 0x20000000, LENGTH = 128K
                 FLASH  (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K
             }
             ```
             这段代码明确通知链接器：
              - 这颗芯片的 Flash 从物理地址 0x08000000 开始，容量只有 1024KB，属性是可读、可执行（rx）。
              - 这颗芯片的 SRAM 从物理地址 0x20000000 开始，容量只有 128KB，属性是可读、可写、可执行（xrw）。
              如果你的代码体积超过了 1024KB，或者变量总大小超过了 128KB，链接器在这一步就会直接报错拦截，防止溢出。
         2. 段（Section）的存放与搬运规则 (SECTIONS)
             - C 语言编写的代码会被编译器拆解为不同的“段”：
               - .text 段：存放编译出来的 CPU 机器指令（所有的函数逻辑）。
               - .rodata 段：存放只读常量（如 const int a = 10; 或双引号字符串常量）。
               - .data 段：存放有初始值的全局变量（如 int count = 100;）。
               - .bss 段：存放未初始化或初始为 0 的全局变量（如 int buffer[1024];）。
            - 链接脚本负责为这些段规划路线：
                - 代码段与常量段（.text、.rodata）：单片机掉电不能丢失代码，所以链接脚本指示将它们放入 FLASH。
                - 中断向量表（.isr_vector）：Cortex-M4 硬件规定复位后必须先去 0x08000000 取栈顶指针和复位地址，所以链接脚本把 .isr_vector 强行排在 FLASH 的最开头。
                - 数据段（.data）的特殊安排：
                    - 全局变量 count 的初值 100 掉电不能丢失，所以初值必须烧录在 FLASH 内部保存。
                    - 但程序运行期间 count 可以随时被修改，而 Flash 无法直接当作内存写，变量必须在 RAM 里运行。
                    - 因此，链接脚本指示：.data 运行地址定在 RAM，但加载存储地址定在 FLASH（>RAM AT> FLASH）。启动汇编代码 startup_stm32f405xx.s 在开机瞬间，就会根据链接脚本标出的 _sdata 与 _sidata 地址，把初值从 Flash 搬运一份到 RAM 中。
4. 完整的编译运行链条
    - 整个工具链的协作顺序如下：
      1. 编译器 (GCC / AS)：把 main.c、hal.c、startup.s 各自分别翻译成目标机器码片段（.o）。
      2. 链接脚本 (.ld)：提供 STM32F405 的 Flash/RAM 空间尺寸与段排布规则。
      3. 链接器 (LD)：参考链接脚本的规则，把所有 .o 组装成一个完整的可执行文件（.elf），并把每个函数和变量分配到具体的绝对内存地址。
      4. 烧录工具 (OpenOCD)：抓取可执行文件，通过 SWD 接口直接把 Flash 镜像刷写入 STM32F405 的 0x08000000 地址空间中。
5. 编写 STM32 链接脚本（Linker Script，后缀通常为 .ld）
   1. 链接脚本的核心框架
   -  一份完整的 ARM Cortex-M 链接脚本由以下四个核心部分构成：
      1. 入口声明 (ENTRY)：指明芯片复位后从哪里开始执行。
      2. 符号定义 (Symbols)：计算栈顶指针、最小堆栈保留大小。
      3. 内存区域宣告 (MEMORY)：定义芯片物理 Flash 与 RAM 的基地址与大小。
      4. 段分配规则 (SECTIONS)：规划各个代码段、数据段分别放进哪块内存。
   1. 声明程序入口与堆栈尺寸
         ```
         /* 1. 指定程序入口点，该符号定义在启动汇编 startup_stm32f405xx.s 中 */
         ENTRY(Reset_Handler)

         /* 2. 定义堆和栈的最小保留空间（单位：字节） */
         _Min_Heap_Size  = 0x200;  /* 堆空间：512 字节（给 malloc 使用） */
         _Min_Stack_Size = 0x400;  /* 栈空间：1024 字节（给局部变量与函数调用使用） */
         ```
   2. 宣告硬件物理内存 (MEMORY)
      - 查阅 STM32F405VGT6 数据手册，其内部物理存储器参数如下：
        - FLASH：容量 1024KB（1MB），物理起始地址为 0x08000000。
        - SRAM：主 SRAM 容量 128KB，物理起始地址为 0x20000000。
        - CCM RAM：核心专用高速 RAM 容量 64KB，物理起始地址为 0x10000000。
      - 在脚本中用 MEMORY 块如实向链接器报备：
         ```
         MEMORY
         {
         /* 属性说明：
             r: 可读 (Read)
             w: 可写 (Write)
             x: 可执行 (Execute)
         */
         FLASH  (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K
         RAM    (xrw) : ORIGIN = 0x20000000, LENGTH = 128K
         CCMRAM (xrw) : ORIGIN = 0x10000000, LENGTH = 64K
         }

         /* 3. 计算栈顶指针地址：SRAM 的末尾（Cortex-M 的堆栈是满递减栈，从高地址往低地址增长） */
         _estack = ORIGIN(RAM) + LENGTH(RAM);
         ```
   3. 规划段分布规则 (SECTIONS)
   - 编译器会将 C 语言代码编译成不同的“段”，我们必须在这里指定它们的去处：
       1. 中断向量表 (.isr_vector)
         汇编启动文件 startup_stm32f405xx.s 中就用 .section .isr_vector 定义了这个表格。
         Cortex-M 内核硬性规定：开机复位时，硬件必须首先读取 Flash 绝对首地址 0x08000000 的内容。因此中断向量表必须在 Flash 的最前方：
             ```
             SECTIONS
             {
                 .isr_vector :                 /* 段的名称 */
                 {
                     . = ALIGN(4);             /* 4 字节地址对齐 */
                     KEEP(*(.isr_vector))      /* KEEP 关键字防止未引用的向量表被优化删除, * 是通配符，代表所有被编译出来的 .o 文件,即收集所有目标文件里面名字叫 .isr_vector 的内容，塞到当前位置 */
                     . = ALIGN(4);
                 } >FLASH
             ```
          > 通俗总结：请在 Flash 的最开头开辟一个叫 .isr_vector 的房间。先确认当前地址是 4 字节对齐的，然后把启动文件里的中断向量表完完整整塞进来。记住，开启代码优化时千万不能把它删掉，最后离开房间前再做一次 4 字节对齐。
       2. 代码段 (.text) 与常量只读段 (.rodata)
         所有由 C 语言编译出的机器指令以及只读常量（如 const 数组、字符串字面量）都要永久保存在 Flash 中：
             ```
                 /* 程序指令段 */
                 .text :
                 {
                     . = ALIGN(4);
                     *(.text)                  /* 收集所有目标文件的 .text 段 */
                     *(.text*)                 /* 收集形如 .text.func 的子段，我们编译时通常会开启 -ffunction-sections 参数（给每个函数独立起一个名字，比如 main() 函数的代码段会被单独命名为 .text.main） */
                     *(.glue_7)                /* 兼容 ARM/Thumb 切换胶水代码 */
                     *(.glue_7t)
                     *(.eh_frame)              /* 异常处理帧（Exception Handling Frame），主要用于 C++ 的异常捕获（try-catch）或者程序崩溃时打印函数调用栈（Stack Unwind） */

                     KEEP (*(.init))           /* 全局构造函数调用入口 */
                     KEEP (*(.fini))           /* 负责程序退出时调用析构函数 */

                     . = ALIGN(4);
                     _etext = .;               /* 记录代码段结束的地址，供后续逻辑参考 */
                 } >FLASH

                 /* 只读常量段 */
                 .rodata :
                 {
                     . = ALIGN(4);
                     *(.rodata)                /* 收集所有常量数据 */
                     *(.rodata*)
                     . = ALIGN(4);
                 } >FLASH
             ```
          > 通俗总结：
             .text：把你写的所有函数装进 Flash。
             .rodata：把你写的所有的只读常量装进 Flash。
       3. 初始化数据段 (.data)
          - 带有初始值的全局变量（例如 int a = 10;），其初始值“10”在断电时不能丢，必须刻录在 Flash 中。
          - 程序运行时这个变量随时会被修改，所以它实际运行的地址必须位于 RAM。
          > 语法 >RAM AT> FLASH ：告诉链接器运行地址在 RAM，但物理烧录存储在 FLASH 中。启动汇编代码 startup_stm32f405xx.s 在开机时，正是读取 _sidata、_sdata 与 _edata，执行循环将数据从 Flash 完整搬运至 RAM 中。
             ```
                 /* 记录数据段在 Flash 中的加载物理首地址（LMA） */
                 /* _sidata：全称是 Start of Initialized Data（初始化数据起始地址）。 */
                 _sidata = LOADADDR(.data);    /* 读取 .data 段在烧录时的物理加载地址（LMA）。此时它紧跟在代码段后面，位于 Flash 中。 */

                 .data : 
                 {
                     . = ALIGN(4);
                     _sdata = .;               /* 导出数据段在 RAM 中的起始运行地址（VMA）,此时它的值就是 RAM 的首地址 */
                     *(.data)                  /* 通配符收集所有 .o 文件里的带初值全局变量，依次排队放好。 */
                     *(.data*)
                     . = ALIGN(4);
                     _edata = .;               /* 导出数据段在 RAM 中的结束运行地址 */
                 } >RAM AT> FLASH
             ```
       4. 未初始化数据段：.bss
             ```
                 .bss :
                 {
                     . = ALIGN(4);             /* 4 字节地址对齐 */
                     _sbss = .;                /* 导出符号：bss 段在 RAM 中的起始地址 */
                     __bss_start__ = _sbss;    /* 提供别名，兼容不同的底层库标准 */
                     *(.bss)                   /* 收集所有源文件的 .bss 段 */
                     *(.bss*)                  /* 收集形如 .bss.var 的子段 */
                     *(COMMON)                 /* 收集弱符号或未决全局变量 */
                     . = ALIGN(4);             /* 4 字节地址对齐 */
                     _ebss = .;                /* 导出符号：bss 段在 RAM 中的结束地址 */
                     __bss_end__ = _ebss;      /* 提供别名 */
                 } >RAM
             ```
             > 启动汇编代码开机时会读取 _sbss 到 _ebss，将这段 RAM 内存统一逐字节清零。
       5. 堆栈空间检查 (._user_heap_stack)
          - 堆（Heap）：用户调用 malloc() 或 free() 时动态分配内存的地方。紧跟在 .bss 数据段后面，由低地址向高地址向上生长。每次调用 malloc()，堆顶指针就增加。
          - 栈（Stack）：函数调用时保存局部变量、中断现场压栈保护的地方。从 RAM 最顶端（_estack = 0x2002 0000）开始，由高地址向低地址向下生长。每次调用函数、声明局部变量或产生中断，栈指针（SP）就减小。
             ```
                 ._user_heap_stack :
                 {
                     . = ALIGN(8);             /* 8 字节对齐（ARM AAPCS 标准规定栈顶必须 8 字节对齐） */
                     PROVIDE ( end = . );      /* PROVIDE 的含义是“按需提供”。end/_end兼容不同版本的 C 标准运行库 */
                     PROVIDE ( _end = . );     /* 当你在代码里第一次调用 malloc() 时，C 标准库会读取 _end，并把这个地址作为动态内存分配的起始底地址。 */
                     . = . + _Min_Heap_Size;   /* 将内存游标强制向后推 512 字节，留给堆 */
                     . = . + _Min_Stack_Size;  /* 将内存游标继续向后推 1024 字节，留给栈 */
                     . = ALIGN(8);
                 } >RAM
             }   
             ```                       
          > 链接器，请帮我算一下：全局变量 + 堆底线 + 栈底线 的总尺寸，有没有把整个 RAM 撑爆
## 五、启动文件
### 了解汇编指令
```assembly
  .syntax unified
  .cpu cortex-m4
  .fpu softvfp
  .thumb
```
#### `.syntax unified`

命令汇编器使用统一汇编语言（Unified Assembler Language，简称 UAL）语法来解析代码。
在早期的 ARM 处理器上，代码存在两种完全分裂的汇编语法（称为 Divided 语法）：
- ARM 状态语法：指令全部是 32 位的，语法要求非常严格。如果你希望指令更新标志位，你必须显式写出 S 后缀（例如 ADDS）。
- 旧版 Thumb 状态语法：指令只有 16 位的，语法规则和 ARM 状态并不通用。很多基础算术指令会默认自动修改标志位，程序员不能随意省略操作数。
- 后来 ARM 公司推出了 Thumb-2 技术（STM32 采用的 Cortex-M 架构正是基于 Thumb-2）。Thumb-2 混合了 16 位和 32 位指令。如果继续使用两套分裂的语法，程序员写汇编代码会极其混乱。
- 为了解决这个问题，ARM 公司设计了 UAL（统一汇编语言）。它让 ARM 指令和 Thumb 指令共用同一套书写规则。.syntax unified 就是开启这套新规则的开关。

---
#### `.cpu cortex-m4 `
- 圈定硬件指令集范围：不同 ARM 内核支持的指令集合不同。声明 cortex-m4 后，汇编器会开启该内核特有的指令支持（如硬件除法指令 UDIV/SDIV、单周期乘加指令以及饱和运算 DSP 指令）。
- 拦截非法指令：如果在代码中写了 Cortex-M4 不支持的指令（例如属于 Cortex-A 架构的虚拟内存管理指令），汇编器在编译阶段就会直接报错，防止生成无法在硬件上执行的非法二进制码。
- 协调管道优化：汇编器会根据 Cortex-M4 的 3 级流水线特性，对分支跳转和指令对齐进行合理的底层处理。

---
#### `.fpu softvfp`
- 告诉汇编器：“当前代码不要生成硬件浮点指令，所有浮点运算都使用纯软件函数来模拟。”,`softvfp`就是 `Software Vector Floating Point（软件向量浮点模拟）`的缩写。
当你在 C 语言里写小数加法时（例如 float c = a + b;），处理器有两种处理方式：
  1. 硬件浮点（Hard-VFP）：
    - 芯片内部有一块专门计算小数的硬件电路（FPU，浮点运算单元）。
    - CPU 直接执行一条专门的硬件指令（如 VADD.F32），几个时钟周期就能算出结果。
  2. 软件模拟浮点（Soft-VFP）：
    - 处理器把小数当作普通的 32 位整型数字。
    - CPU 无法直接执行浮点硬件指令，而是通过调用一段由几十条整数加减移位指令组成的软件函数（如 __aeabi_fadd）来强行模拟出浮点结果。
    - 这种方式计算速度较慢，但它对硬件完全没有要求，任何基础 ARM 处理器都能运行。
- STM32F405 明明自带硬件 FPU，为什么启动文件还要写 softvfp
    1. 硬件 FPU 开机默认是关闭的：
        - ARM 架构规定：芯片复位通电时，为了降低功耗和确保安全，硬件 FPU 默认处于关闭断电状态（Disabled）。
        - 如果此时 CPU 贸然执行任何硬件浮点指令（比如以字母 V 开头的指令 VLDR、VADD），处理器会因为硬件未启用而立即触发 UsageFault（未定义指令异常），直接导致单片机开机死机。
    2. FPU 的开启工作在后续代码中才完成：
        - 开启 FPU 的寄存器配置代码写在 SystemInit() 函数中：
        ```
        /* 位于 system_stm32f4xx.c 中，向 CPACR 寄存器写入配置以启用 FPU */
        SCB->CPACR |= ((3UL << 10*2) | (3UL << 11*2));
        ```
        - 启动汇编运行在 SystemInit() 之前。在启动文件运行的这段时间里，硬件 FPU 还没有通电。
    3. 防止汇编器生成非法指令：
        - 在启动文件开头加上 .fpu softvfp，等于向汇编器加上了一道安全锁：在启动阶段，严禁生成任何硬件浮点指令，从而确保代码在 FPU 开启之前绝对安全、稳定地运行。

---
#### `.thumb`
- 这条指令命令汇编器生成 Thumb / Thumb-2 编码格式的机器指令，而不是传统的 32 位 ARM 编码指令。
- 在老款 ARM 芯片（如 ARM7TDMI）时代：
芯片只有两种状态：32 位的 ARM 状态 和 16 位的 Thumb 状态。
当时的汇编器提供了两个切换指令：
  - .arm：生成 32 位 ARM 代码。
  - .thumb：生成 16 位 Thumb 代码。
- 后来 ARM 推出了 Thumb-2 技术（混合 16 位和 32 位指令的技术）。Thumb-2 并不是一个独立的全新模式，而是对原有 Thumb 模式的向后兼容扩展。
- 为了不破坏已有的编译器习惯和脚本工具链，GNU 汇编器（GAS）做了一个规则调整：
- 只要你在前面指定了支持 Thumb-2 的现代内核（比如 .cpu cortex-m4）：
此时汇编器只要看到 .thumb，就会自动开启 完整的 Thumb-2 指令集支持（既能编译 16 位指令，也能编译 32 位 Thumb-2 宽指令）。
因此： .cpu cortex-m4 与 .thumb 配合在一起，让它具备了 Thumb-2 的能力。
- Thumb-2 采用 16 位与 32 位混合编码。在保持高性能的同时，能够节省 30% 左右的 Flash 占用空间。

---
```
.global  g_pfnVectors
.global  Default_Handler
```
#### `.global`
- 告诉汇编器把紧随其后的符号导出为全局符号，类比 C 语言里不加 static 的全局变量。
- 汇编文件中定义的符号（标号）默认只有当前文件内部可见。
    - g_pfnVectors 是中断向量表（那个装满函数地址的数组）的名字，必须被链接脚本和 C 语言代码访问。链接脚本通过符号名称把向量表精确放置在 Flash 起始位置；C 代码在重映射向量表时也需要通过 extern void *g_pfnVectors[]; 读取它。
    - Default_Handler 需要被后续所有未定义中断弱符号绑定（.thumb_set），将其设为全局符号后，链接器在多目标文件合并时才能正确识别。Default_Handler 是一个“备用接线员”（里面是一个死循环 while(1)）。
        ```
        .thumb_set USART1_IRQHandler, Default_Handler
        .thumb_set TIM2_IRQHandler,   Default_Handler
        .thumb_set ADC_IRQHandler,    Default_Handler
        ```
        - 这段代码的意思是：“如果用户在 C 语言里没写 USART1_IRQHandler，那么发生串口中断时，全部统一转接到 Default_Handler 处理！”
        - 在实际编译时，由于开启了函数分段编译优化，每个中断和函数都会被拆成独立的符号。如果 Default_Handler 不加 .global，链接器在把其他文件的引用转接到这里时，就会发现这个备用接线员是“非公开的隐藏人员”，从而导致符号解析失败。

---
```
/* start address for the initialization values of the .data section. defined in linker script */
.word  _sidata
/* start address for the .data section. defined in linker script */  
.word  _sdata
/* end address for the .data section. defined in linker script */
.word  _edata
/* start address for the .bss section. defined in linker script */
.word  _sbss
/* end address for the .bss section. defined in linker script */
.word  _ebss
```
#### `.word`
- 意思是在当前内存位置塞入一个 32 位（4 字节）的数据，STM32（ARM Cortex-M）是 32 位单片机，它的一个标准“字”（Word）占用 4 个字节（32 位）。
- 当你在汇编里写下：.word 0x12345678，汇编器就会在生成的二进制文件当前游标处，实实在在地塞进 4 个字节：78 56 34 12（ARM 默认小端模式存储）。
- 这 5 行代码在只读内存中连续开辟了 5 个 32 位（4 字节）存储单元，保存了链接脚本计算出的 5 个关键边界地址。
  - _sidata（Start of Initialized Data）：
  存放在 Flash 中的全局变量初始值烧录基地址（LMA，加载存储地址）。
  - _sdata（Start of Data）：
  位于 RAM 中的全局变量运行时起始地址（VMA，虚拟内存地址）。
  - _edata（End of Data）：
  位于 RAM 中的全局变量运行时结束地址。
  - _sbss（Start of BSS）：
  位于 RAM 中的未初始化全局变量起始地址。
  - _ebss（End of BSS）：
  位于 RAM 中的未初始化全局变量结束地址。

---
#### `.section`
- 可以把.section认为是一个箱子，.section后面跟着的就是这个箱子的名字，后面的东西都装进箱子里；
在 GNU ARM 汇编（GAS 语法）中，.section 的标准语法格式如下：
```
.section name [, "flags" [, %type [, entsize]]]
```
方括号 [ ] 内的内容代表可选参数。
   - name（段名称，必填）
       - 通用标准段名：.text（代码）、.data（有初值数据）、.bss（无初值数据）、.rodata（只读数据）。
       - 自定义子段名：通过加点扩展，比如 .text.Reset_Handler、.isr_vector。链接脚本会通过通配符（如 *(.text*)）将它们收集起来。
   - "flags"（权限标志，可选，双引号包裹）
       - a（Allocatable）：必须占用物理内存（Flash 或 RAM）。
       - w（Writable）：运行时内容允许被 CPU 写入修改。
       - x（Executable）：内容包含机器指令，允许 CPU 跳转执行。
       - s（Shared）：多核或共享内存段（单片机极少用）。
   - %type（内容类型，可选，以 % 或 @ 开头）告诉汇编器这个段里面装的是实实在在的内容，还是空占位符：
       - %progbits（Program Bits）：段内包含真实的程序二进制内容（指令、常量、地址表格）。链接器生成烧录镜像时会把这些数据实实在在刻录进去。
       - %nobits：段内不含实际二进制数据，仅仅是一个大小占位符（最典型的就是 .bss 段）。生成烧录镜像时不占 Flash 空间。
   - entsize（表项大小，极少用）
       如果该段是一个固定大小结构体的数组（例如包含特定合并项），可以传入每个元素的字节大小，通常省略不写。
> 在汇编代码中，经常会看到参数完整度和形式不同的写法：
> ```
> /* 1. 完整严谨写法：明确告诉工具链其段名、权限与类型 */
> .section .isr_vector, "a", %progbits
> .section .text.Default_Handler, "ax", %progbits
> 
> /* 2. 简写形式：利用汇编器的内建规则 */
> .section .text.Reset_Handler
> ```
> 汇编器的自动推断规则：
> 如果省略 "flags" 和 %type，汇编器会根据段名的开头前缀进行自动推断：
> - 凡是以 .text 开头的段，默认补齐为 "ax", %progbits。
> - 凡是以 .data 开头的段，默认补齐为 "aw", %progbits。
> - 凡是以 .bss 开头的段，默认补齐为 "aw", %nobits。
> - 凡是以 .rodata 开头的段，默认补齐为 "a", %progbits。
> - 自定义名称（如 .isr_vector）：汇编器无法推断意图，必须显式写出 "a", %progbits，否则工具链可能不会把它当作合法的只读数据段处理。

---
#### `ldr（Load Register，内存读入寄存器）`
从指定的内存地址读取 4 字节（32 位）数据，写入目标通用寄存器。
常见语法格式：
- 伪指令加载 32 位常数/地址：`ldr <Rd>, =expr`
  - 示例：`ldr r0, =_sdata`
  - 作用：把符号 _sdata 代表的 32 位内存地址一次性装入寄存器 r0（汇编器会自动通过 PC 相对寻址从文字池中取出）。
- 基址寻址：`ldr <Rd>, [<Rn>]`
  - 示例：`ldr r4, [r2]`
  - 作用：读取寄存器 r2 所存地址处的 4 字节数据，存入 r4。
- 基址加偏移量寻址：`ldr <Rd>, [<Rn>, <Rm>]`
  - 示例：`ldr r4, [r2, r3]`
  - 作用：读取地址为 (r2 + r3) 处的内存数据，存入 r4。
---
#### `str（Store Register，寄存器写出到内存）`
将寄存器中的 4 字节数据，写到指定的目标内存地址中。
常见语法格式：
- 基址寻址：`str <Rd>, [<Rn>]`
  - 示例：`str r4, [r0]`
  - 作用：将 r4 的数据写入 r0 所指向的内存单元。
- 基址加偏移量寻址：`str <Rd>, [<Rn>, <Rm>]`
  - 示例：str r4, [r0, r3]
  - 作用：将 r4 的数据写入 (r0 + r3) 所对应的内存单元。
> 记忆技巧：
> - ldr 是“从右往左进寄存器”（从内存 Load 进 CPU）。
> - str 是“从左往右出到内存”（从 CPU Store 到内存）。

---
#### `movs（Move with Flags，立即数/寄存器赋值并更新标志位）`
将一个常数或另一个寄存器的值传送到目标寄存器，
后缀 s 表示强制更新 APSR（应用程序状态寄存器）的条件标志位（如 N、Z 标志）。
语法格式：
- `movs <Rd>, #<imm8>`
- `movs <Rd>, <Rm>`
代码示例：
    ```
    movs r3, #0      /* 将立即数 0 写入 r3，同时因为结果为 0，Z (Zero) 标志位会被置 1 */
    ```

---
#### `adds（Add with Flags，加法并更新标志位）`
对两个操作数执行加法运算，将结果存入目标寄存器，后缀 s 表示运算后根据结果刷新进位（C）、溢出（V）、为零（Z）、为负（N）等标志位。
常见语法格式：
- 寄存器加立即数：`adds <Rd>, <Rn>, #<imm> `或 `adds <Rd>, #<imm>`
  - 示例：`adds r3, r3, #4`（让偏移量 r3 增加 4 字节，等价于 C 语言的 r3 += 4）。
- 两寄存器相加：`adds <Rd>, <Rn>, <Rm>`
  - 示例：adds r4, r0, r3（计算 r4 = r0 + r3，常用于算出当前绝对访问地址）。

---
#### `cmp（Compare，大小/数值比较）`
比较两个操作数的数值关系。它的底层实现本质是执行无结果保存的减法运算（执行 `Rn - Rm 或 Rn - #imm`），差值被直接丢弃，专门用来刷新 CPU 状态寄存器中的标志位。
语法格式：
- `cmp <Rn>, <Rm>`
- `cmp <Rn>, #<imm>`
代码示例：
```
cmp r4, r1       /* 硬件计算 r4 - r1，更新 C、Z、N 等标志位 */
```

---
#### `bcc（Branch if Carry Clear，无进位/小于则跳转）`
根据状态寄存器中的进位标志位（C 标志）决定是否执行分支跳转。
核心逻辑：
- 在 ARM 架构中，做减法比较（cmp A, B）时：
  - 若 $A \ge B$（无借位），进位标志位被置 $1$（Carry Set）。
  - 若 $A < B$（产生借位），进位标志位保持 $0$（Carry Clear）。
- 因此，`bcc <label> `在无符号数比较后，完全等价于 blo（Branch if Lower，若小于则跳转）。
代码示例：
```
cmp r4, r1       /* 比较当前指针 r4 与边界 r1 */
bcc CopyDataInit /* 若 r4 < r1，则跳回 CopyDataInit 继续执行循环 */
```

---
#### `.weak 伪指令`
.weak 用于将一个符号标记为弱符号（Weak Symbol）。
1. 基本语法

    ```.weak <symbol_name>```
   - <symbol_name>：需要修饰的函数名或全局变量名。
2. 底层原理与核心作用
在链接过程中，符号默认都是强符号（Strong Symbol）。
   - 默认规则：如果链接器在多个源文件中发现了同名的强符号，链接器会直接报错：multiple definition of 'xxx'（符号重复定义）。
   - 弱符号规则：
   1. 强弱共存，强符号优先：如果程序在其他地方定义了一个同名的强符号，链接器会自动选用强符号，并忽略此处的弱符号。
   2. 多个弱符号：如果程序中只有多个同名弱符号，链接器会挑选其中一个，不会报错。
   3. 未定义也不报错：弱符号即使在整个工程中完全没有实体定义，链接时它的地址会被默认解析为 0，而不会报 undefined reference 错误。

3. 典型代码示例
在 STM32 启动文件中，中断函数和复位函数普遍使用 .weak：
    ```
    /* 将 Reset_Handler 声明为弱符号 */
    .weak Reset_Handler
    Reset_Handler:
        ldr sp, =_estack
        ...
    ```
实际效果：
启动文件提供了一套默认的 Reset_Handler 汇编实现。
如果你在自己的 C 代码或 RTOS 内核中重新写了一个 void Reset_Handler(void)，链接器会自动用你写的强符号覆盖掉启动文件里的默认实现，且不会发生重复定义冲突。

---
#### `.type 伪指令`
.type 用于明确告诉汇编器和链接器：这个符号的物理实体究竟是一个函数（可执行代码），还是一个对象（数据变量）。
1. 基本语法
    ```
    .type <symbol_name>, %<type_description>
    /* 或者使用 @ 符号（两者等价）： */
    .type <symbol_name>, @<type_description>
    ```
2. 支持的常见类型参数
   - %function：声明该符号是一个可执行函数（代码入口）。
   - %object：声明该符号是一个数据对象（变量、常量或数组）。
   - %notype：未指定具体类型（汇编默认缺省状态）。

3. 为什么在 ARM Cortex-M 中必须声明 %function？
   1. Thumb 模式切换位（Bit 0 置 1）：
      - ARM Cortex-M 处理器只支持 Thumb / Thumb-2 指令集，不支持 ARM 模式。
      - ARM 架构规定：跳转到 Thumb 代码执行时，目标地址的最低位（Bit 0）必须为 1。
      - 当你为标号加上 `.type Symbol, %function` 后，汇编器和链接器就会知道它是一个函数。在其他地方引用这个函数的地址时（比如中断向量表里的 .word Reset_Handler），工具链会自动把地址值加 1（即 Bit 0 置 1）。
      - 如果漏掉了 %function，工具链可能会将其当作纯数据地址（Bit 0 为 0）。CPU 跳转过去时就会因为最低位未置 1 而直接触发硬件异常（HardFault）。
   2. 调试器识别调用栈：
      - GDB 调试器依赖 ELF 符号表里的类型标记。被标记为 %function 的符号在反汇编和单步调试时会被识别为独立的函数栈帧，便于查看函数调用关系。

---
#### `.size`
1. 基本语法格式
    ```
    .size <name>, <expression>
    ```
    - `<name>`（必填）：需要指定大小的符号名称（例如函数名、数组名或变量标号）。
    - `<expression>`（必填）：表示该符号所占字节总数的算术表达式。
2. 核心用法与经典表达式
   1. 动态自动计算大小（最标准、最常见的写法）
   在编写函数或数组时，我们通常不想手动去数它到底占了多少字节。在汇编中，我们使用 `.-<name>` 来实现自动计算：
       ```
       Reset_Handler:
       ldr   sp, =_estack
       bl    SystemInit
       bl    main
       bx    lr
       .size Reset_Handler, .-Reset_Handler
       ```
      - .（当前位置计数器）：代表汇编器处理到当前这一行时的结束物理地址。
      - Reset_Handler：代表该函数入口的第一行指令所在的起始物理地址。
      - .-Reset_Handler：两者相减，直接得出该函数从头到尾的实际总字节数。
   2. 固定常数大小
   对于已知长度的变量或固定表格，也可以直接传入具体的数字：
        ```
        my_var:
            .word 0x12345678
            .size my_var, 4         /* 明确指定 my_var 占用 4 字节 */
        ```
3. 为什么必须使用 .size？
如果不写 .size，程序烧录进单片机依然可以运行，但是在调试与分析时会出现诸多不便：
   1. 协助 GDB 调试器识别函数边界：
      - GDB 依靠 ELF 文件中的尺寸信息来框定函数范围。
      - 如果没有 .size，调试器只知道函数从哪里开始，不知道在哪里结束。在单步调试、反汇编查看或打印调用栈（Stack Backtrace）时，调试器可能会把后面的代码或数据误判为当前函数的一部分。
   2. 辅助静态分析与内存统计工具：
      - 执行 arm-none-eabi-size 或 arm-none-eabi-nm -S 查看固件占用时，工具依赖 .size 输出每个函数与全局变量的精确字节开销。
   3. 辅助高级链接优化：
      - 某些高级链接器优化（如针对函数段的重排和分析）需要准确掌握每个符号的尺寸边界。

4. 标准代码搭配结构
在 Cortex-M 的启动文件与底层驱动汇编中，一个标准函数的定义通常遵循“头尾呼应”的结构：

    ```
    .section .text.Reset_Handler
    .weak    Reset_Handler
    .type    Reset_Handler, %function   /* 1. 声明符号为函数 */
    Reset_Handler:                         /* 2. 标记函数起始入口 */
    ldr   sp, =_estack
    bl    SystemInit
    bl    main
    bx    lr
    .size Reset_Handler, .-Reset_Handler /* 3. 标记函数结束并结算大小 */
    ```

---
#### `bl（Branch with Link，带返回链接的跳转）`
1. 核心作用
bl 是执行函数调用的标准指令。它等价于 C 语言里的 func();。
2. 底层执行动作（做两件事）
当 CPU 执行 bl SystemInit 时，硬件在同一个时钟周期内自动完成两步：
   1. 保存返回地址到 LR 寄存器：
   CPU 将紧随其后的下一条指令地址写入链接寄存器 LR（Link Register，即寄存器 R14）。

   2. 跳转到目标函数：
   CPU 将目标函数的入口地址写入程序计数器 PC（Program Counter，即寄存器 R15），随后开始执行被调用的函数。

3. 代码示例
```
/* 启动文件中的典型调用 */
bl  SystemInit     /* 1. LR 保存下一行代码地址；2. PC 跳入 SystemInit 执行 */
bl  main           /* 1. LR 保存下一行代码地址；2. PC 跳入 main 执行 */
```

---
#### ` bx（Branch and Exchange，寄存器间接跳转并切换状态）`
1. 核心作用
bx 用于根据寄存器里的地址进行跳转，并且在跳转的同时根据目标地址的最低位（Bit 0）检查或切换处理器状态。在绝大多数情况下，它用于从子函数返回（等价于 C 语言的 return;）。
2. 最经典的用法：`bx lr`（函数返回）
   - 函数执行到最后一行时，执行 bx lr。
   - CPU 把之前在 LR 寄存器里保存的“返回地址”重新倒回给 PC。
   - 程序立刻精准返回到当初调用该函数的下一行代码继续执行。
3. 为什么名字里带“Exchange（交换/切换）”？
ARM 架构支持 ARM 状态（32位指令）与 Thumb 状态（16/32位指令）。bx 会自动检查目标寄存器的 Bit 0（最低位）：
   - 如果最低位是 1：CPU 保持/切换为 Thumb 状态。
   - 如果最低位是 0：CPU 保持/切换为 ARM 状态。
在 STM32（Cortex-M 架构）中，硬件只支持 Thumb 指令。因此所有函数地址的最低位在编译时都会被自动置为 1。执行 bx 时，硬件既能准确跳到偶数地址执行指令，又能维持合法的 Thumb 运行状态。
4. 经典配合流程：函数的“去”与“回”
在底层汇编中，bl 与 bx lr 永远是成对配合工作的：
    ```
    调用者代码（Caller）：
    bl MyFunction     ───┐ (1. 将下行地址存入 LR，跳入 MyFunction)
    movs r0, #0       <──┼──────────────────────┐
                         │                      │
    子函数代码(Callee)    │                      │
    MyFunction:          │                      |
    adds r1, r2, #1   <──┘                      │
    bx lr             ──────────────────────────┘ (2. 读取 LR 地址写入 PC，返回)
    ```

##### 启动文件是芯片上电复位后执行的第一段程序。

整个启动文件由四个部分组成:
1. 建立中断向量表 (g_pfnVectors) , 填入栈顶指针和所有中断处理函数地址
    ```
    .section .isr_vector,"a",%progbits
    .type g_pfnVectors, %object
    ```
    - .type：用来指定一个符号的类型。
    - g_pfnVectors：符号的名字（g_ 表示全局，pfn 表示函数指针，Vectors 表示向量）。
    - %object：告诉编译器与调试器，g_pfnVectors 是一个数据对象（变量/数组），而不是一段可以直接跳转执行的代码函数（如果是函数，这里会写 %function）。
    ```
    .size g_pfnVectors, .-g_pfnVectors
    ```
    - .size：用来计算并声明该符号在内存中占用的总字节大小
    - .（当前地址）：表示当前这一行的内存地址游标。
    - g_pfnVectors：表示整个数组起始位置的内存地址。
    - .-g_pfnVectors：这是汇编里的指针差值计算。用当前游标地址减去起始地址，算出的结果正好就是整张向量表占用的字节总数。调试器可以通过这个属性准确知道这个数组有多大。
    ```
    .word _estack
    ```
    - .word：伪指令，意思是在当前内存位置塞入一个 32 位（4 字节）的数据，STM32（ARM Cortex-M）是 32 位单片机，它的一个标准“字”（Word）占用 4 个字节（32 位）。当你在汇编里写下：.word 0x12345678，汇编器就会在生成的二进制文件当前游标处，实实在在地塞进 4 个字节：78 56 34 12（ARM 默认小端模式存储）。它背后的本质：这里的 Reset_Handler、NMI_Handler 都是函数名字。在汇编里，直接写函数名代表获取该函数的绝对内存地址。
    ```
    .section .isr_vector,"a",%progbits
    .type g_pfnVectors, %object
    .size g_pfnVectors, .-g_pfnVectors

    g_pfnVectors:            /* 冒号前面是标号（Label）。它相当于 C 语言里的数组名，代表向量表的起始地址 */
    .word _estack            /* 0号表项：栈顶地址 */
    .word Reset_Handler      /* 1号表项：复位函数地址 */
    .word NMI_Handler        /* 2号表项：不可屏蔽中断函数地址 */
    .word HardFault_Handler  /* 3号表项：硬件错误异常 */
    ...
    .word SysTick_Handler    /* 系统滴答定时器中断 */
    .word USART1_IRQHandler  /* 串口1中断 */
    ```
2. 复位处理函数 (Reset_Handler)
```
/* 1. 加载内存边界地址到通用寄存器 */
  ldr   r0, =_sdata         /* 将 .data 段在 RAM 中的起始目标地址（VMA）装载到寄存器 r0 */
  ldr   r1, =_edata         /* 将 .data 段在 RAM 中的结束地址装载到寄存器 r1，用作循环退出的边界守卫。 */
  ldr   r2, =_sidata        /* 将 .data 段初值在 Flash 中的原始存放地址（LMA）装载到寄存器 r2 */
  movs  r3, #0              /* 将寄存器 r3 清零。r3 在整个搬运过程中充当字节偏移量计数器（Offset） */
  b     LoopCopyDataInit    /* 无条件直接跳转到末尾的 LoopCopyDataInit 进行边界检查 */

/* 2. 循环搬运主体 */
CopyDataInit:
  ldr   r4, [r2, r3]       /* 从 Flash (r2 + r3) 读取 4 字节数据到 r4 */
  str   r4, [r0, r3]       /* 将 r4 的数据写入 RAM (r0 + r3) */
  adds  r3, r3, #4         /* 偏移量增加 4 字节 */

/* 3. 循环边界判断 */
LoopCopyDataInit:
  adds  r4, r0, r3         /* 计算当前 RAM 写入位置：r4 = _sdata + 偏移量 */
  cmp   r4, r1             /* 比较当前位置与 RAM 结束地址 (_edata) ,本质上是执行 r4 - r1（当前 RAM 写入游标减去 RAM 终止地址 _edata），不保留差值结果，只根据相减结果刷新 CPU 条件标志位*/
  bcc   CopyDataInit       /* 若 r4 < r1 (无借位/进位标志未置位)，继续拷贝,bcc 等价于若小于则跳转 */
```
这段汇编代码完全等价于以下 C 语言指针遍历循环：
```
extern uint32_t _sidata; /* Flash 中初值存放的起始物理地址 */
extern uint32_t _sdata;  /* RAM 中目标起始地址 */
extern uint32_t _edata;  /* RAM 中目标结束地址 */

void Copy_Data_Section(void) {
    uint32_t *flash_src = &_sidata;
    uint32_t *ram_dst   = &_sdata;

    while (ram_dst < &_edata) {
        *ram_dst++ = *flash_src++;
    }
}
```

## 六、核心应用代码与系统配置
#### 硬件总配置文件：stm32f4xx_hal_conf.h
stm32f4xx_hal.h中有
```
/* Includes ------------------------------------------------------------------*/
#include "stm32f4xx_hal_conf.h"
```
如果没有这个配置文件，HAL 库在编译的第一行就会报错说找不到文件。
1. HAL 库通过宏定义来控制编译范围。比如：
    ```
    #define HAL_MODULE_ENABLED
    #define HAL_GPIO_MODULE_ENABLED
    #define HAL_RCC_MODULE_ENABLED
    /* #define HAL_UART_MODULE_ENABLED */
    /* #define HAL_SPI_MODULE_ENABLED */
    ```
2. 向驱动库提供当前板子的“基础物理参数”
不同开发板焊接的晶振频率不同：有的板子接了 8MHz 晶振，有的接了 12MHz 晶振，还有的直接用内部 RC 振荡器。
HAL 库的代码是通用的，它自己不知道你板子上到底贴了多大频率的晶振。因此，必须在 stm32f4xx_hal_conf.h 中如实填入：
    ```
    #define HSE_VALUE    ((uint32_t)8000000U) /* 告诉驱动库：外部高速晶振是 8MHz */
    #define HSI_VALUE    ((uint32_t)16000000U)/* 内部高速振荡器是 16MHz */
    #define VDD_VALUE    ((uint32_t)3300U)    /* 供电电压是 3.3V */
    ```
1. 官方 HAL 库配置模板：
    - 文件路径：STM32CubeF4/Drivers/STM32F4xx_HAL_Driver/Inc/stm32f4xx_hal_conf_template.h
    - 官方规则：ST 官方要求开发者将该模板文件复制到工程的头文件目录Core/Inc/中，并且将文件名修改为 stm32f4xx_hal_conf.h。
#### 中断处理文件： stm32f4xx_it.c / .h
- 在汇编启动文件 startup_stm32f405xx.s 中，中断向量表已经预留了所有的异常与中断入口符号（如 SysTick_Handler、HardFault_Handler）。这两个文件负责在 C 语言层面具体实现这些中断函数，充当 CPU 响应硬件事件后的实际执行入口。
- 在 ARM Cortex-M 架构（包括 STM32F405）中，所有中断和异常统一由 NVIC（嵌套向量中断控制器） 管理，主要分为两类：
    1. 系统内核异常,中断号 1 ~ 15
        - ARM 官方 Cortex-M4 内核自带。无论哪家芯片厂商（ST、NXP、TI），这套异常完全一致。
    2. 片上外设中断,中断号 16 及以上
        - 芯片厂商（ST 公司）围绕内核自主添加的外设硬件。

#### 主函数 main.c？
启动文件（startup_stm32f405xx.s）在开机时完成了段搬运和清零之后，最后执行了一条关键的跳转指令：
```
bl main
```
这行汇编指令指示 CPU：跳入 C 语言的 main 函数开始执行。
1. 