### 一、环境配置
### 二、获取官方固件包源文件
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
### 三、精简并抽取核心依赖
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
    ***
    ###### 具体操作：
    1. 打开解压后的固件包路径：
    STM32CubeF4/Drivers/CMSIS/Include/

    2. 将这个文件夹内的所有 .h 头文件，全部复制到工程的：
    Drivers/CMSIS/Include/ 目录下。
2. 外设寄存器定义（房间门牌号）
   - 外设寄存器映射 (==stm32f405xx.h==)：虽然内核是 ARM 设计的，但外设（GPIO、USART、TIM、SPI 等）是 ST 公司设计的。ST 为STM32F405 的每个寄存器分配了绝对物理地址（例如 GPIOA 的基地址是 0x40020000）。这个头文件通过结构体指针将内存地址映射为 C 语言对象，没有它，代码就无法识别诸如 GPIOA->MODER 这样的写法。
   - 时钟基础配置 (==system_stm32f4xx.c==)：芯片上电时默认运行在慢速的内部 RC 振荡器（HSI 16MHz）上。该文件定义了系统全局变量 SystemCoreClock，并提供了 SystemInit() 函数。该函数主要完成两项任务：
     1. 打开 Cortex-M4 的 CP10 和 CP11 协处理器控制寄存器，开启硬件浮点单元（FPU）。
     2. 确保中断向量表基地址偏移（VTOR）指向正确的 Flash 起始位置。

    ***
    ###### 具体操作：
    1. 提取芯片寄存器定义头文件：
        - 打开固件包路径：STM32CubeF4/Drivers/CMSIS/Device/ST/STM32F4xx/Include/
        - 复制以下 3 个文件到工程的 Drivers/CMSIS/Device/ST/STM32F4xx/Include/：
            - stm32f4xx.h
            - stm32f405xx.h
            - system_stm32f4xx.h
    2. 提取时钟初始化源文件：
        - 打开固件包路径：STM32CubeF4/Drivers/CMSIS/Device/ST/STM32F4xx/Source/Templates/
        - 复制 system_stm32f4xx.c 到工程的 Core/Src/ 目录下。

3. 启动汇编（地基施工队）
   - GNU 启动汇编 (==startup_stm32f405xx.s==)：微控制器上电时不能直接从 main() 函数开始跑。它需要一段汇编代码完成硬件准备：
     1. 定义中断向量表（放在内存起始位置 0x08000000，首个字为堆栈指针顶端 _estack，第二个字为复位入口 Reset_Handler）。
     2. 初始化 C 运行环境：把 Flash 里的常量数据复制到 RAM（.data 段），并把 RAM 里未初始化的全局变量清零（.bss 段）。
     3. 调用 SystemInit()，最后跳转到 main()。
    ***
    ###### 具体操作：
    ##### 因为选择的工具链是 ARM GNU Toolchain (GCC)，所以必须选择 gcc/ 目录下的 startup_stm32f405xx.s。
    1. 提取启动汇编文件：
        - 打开固件包路径：STM32CubeF4/Drivers/CMSIS/Device/ST/STM32F4xx/Source/Templates/gcc/
        - 复制 startup_stm32f405xx.s 到工程的 Core/Src/ 目录下。
4. 硬件驱动库（现成的水暖电模块）
    - HAL 库之间存在较多依赖，但下述 6 个模块是 STM32 运行的“最小生命维持集”。先只保留这几个模块，能避免因为缺少其他外设宏而引发一连串难以排查的编译报错。后续需要串口或定时器时，再随时向里面追加 stm32f4xx_hal_uart.c 等文件即可。
    ***
    ###### 具体操作：
    1. 复制头文件：
       - 打开路径：STM32CubeF4/Drivers/STM32F4xx_HAL_Driver/Inc/
       - 复制以下头文件到工程的 Drivers/STM32F4xx_HAL_Driver/Inc/：
         - stm32f4xx_hal.h（HAL 核心总头文件）
         - stm32f4xx_hal_def.h（HAL 状态码与结构体定义）
         - stm32f4xx_hal_cortex.h（SysTick 定时器与中断封装）
         - stm32f4xx_hal_rcc.h、stm32f4xx_hal_rcc_ex.h（时钟复位控制）
         - stm32f4xx_hal_gpio.h、stm32f4xx_hal_gpio_ex.h（通用输入输出口，用于点灯）
         - stm32f4xx_hal_pwr.h、stm32f4xx_hal_pwr_ex.h（电源管理与超频调压）

    2. 复制对应的 C 源文件：
       - 打开路径：STM32CubeF4/Drivers/STM32F4xx_HAL_Driver/Src/
       - 复制以下 .c 文件到工程的 Drivers/STM32F4xx_HAL_Driver/Src/：
         - stm32f4xx_hal.c（初始化 HAL 计时滴答与内部机制）
         - stm32f4xx_hal_cortex.c
         - stm32f4xx_hal_rcc.c、stm32f4xx_hal_rcc_ex.c
         - stm32f4xx_hal_gpio.c
         - stm32f4xx_hal_pwr.c、stm32f4xx_hal_pwr_ex.c
### 四、编译器与链接脚本
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