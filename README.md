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
    编译器（Compiler）是一个翻译工具。微控制器（MCU）只认二进制机器指令（即 0 和 1 组成的机器码），不懂人类写的 C/C++ 或汇编代码。在工具包中，arm-gnu-toolchain 就是这一套工具的集合：
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