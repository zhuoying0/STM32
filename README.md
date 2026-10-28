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
             .text：把你写的所有函数动作装进 Flash。
             .rodata：把你写的所有死记硬背的只读常量装进 Flash。
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
##### 了解.section：
汇编器工作时，心里永远维护着一个当前状态：“我现在正在往哪个段（Section）里写东西？”
##### 启动文件是芯片上电复位后执行的第一段程序。

整个启动文件由四个部分组成:
1. 建立中断向量表 (g_pfnVectors) , 填入栈顶指针和所有中断处理函数地址
    ```
    .section .isr_vector,"a",%progbits
    ```
    - .section：告诉汇编器，下面开始的内容属于一个特定的内存段,
    - .isr_vector：这个段的自定义名称。我们在链接脚本 .ld 里写的 KEEP(*(.isr_vector))，就是通过这个名字找到这里的。
    - "a"：段的属性标志，代表 allocatable（可分配的）。它通知链接器在程序运行时必须加载进物理内存（占用 Flash 或 RAM）。与此相对的，不写 a 代表Non-allocatable纯辅助信息段（如调试信息、编译器版本注释），仅留在电脑的 ELF 文件中，"x"	Executable	该段包含可执行的机器指令（CPU 可以跳转执行），"w"	Writable	该段在程序运行时允许被写入修改（数据变量）
    - %progbits：段的数据类型说明，代表 Program Bits（程序数据）。它表明该段包含的是实际的程序代码或有效数据，而不是空无一物的占位符（与之相对的是用来占位的 %nobits）。
    
    ```
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
    - .word：伪指令，意思是在当前内存位置塞入一个 32 位（4 字节）的数据。它背后的本质：这里的 Reset_Handler、NMI_Handler 都是函数名字。在汇编里，直接写函数名代表获取该函数的绝对内存地址。
    ```
    .section .isr_vector,"a",%progbits
    .type g_pfnVectors, %object
    .size g_pfnVectors, .-g_pfnVectors

    g_pfnVectors:            /* 冒号前面是标号（Label）。它相当于 C 语言里的数组名，代表向量表的起始地址 */
    .word _estack            /* 0号表项：初始主堆栈指针 (MSP) */
    .word Reset_Handler      /* 1号表项：复位中断服务函数入口 */
    .word NMI_Handler        /* 2号表项：不可屏蔽中断 */
    .word HardFault_Handler  /* 3号表项：硬件错误异常 */
    ...
    .word SysTick_Handler    /* 系统滴答定时器中断 */
    .word USART1_IRQHandler  /* 串口1中断 */
    ```
2. 复位处理函数 (Reset_Handler)
   - 把有初始值的变量从 Flash 搬运到 RAM (.data 段搬运)
   - 把未初始化的变量区域全部刷零 (.bss 段清零)
3. 调用底层时钟初始化 SystemInit() , 开启硬件浮点单元 (FPU) 等
4. 调用 C 语言主入口 main()

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