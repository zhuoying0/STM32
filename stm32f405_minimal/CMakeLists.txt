cmake_minimum_required(VERSION 3.20)

# 1. 声明交叉编译器（必须在 project 命令之前）
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

# 👇【关键修改：添加下面这行，跳过 CMake 的裸机链接检查】👇
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
set(CMAKE_OBJCOPY arm-none-eabi-objcopy)
set(CMAKE_SIZE_UTIL arm-none-eabi-size)

# 2. 定义工程名称与支持语言
project(stm32f405_minimal C ASM)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# 3. 硬件架构与编译选项（Cortex-M4 + 硬件单精度浮点）
set(CPU_FLAGS
    -mcpu=cortex-m4
    -mthumb
    -mfpu=fpv4-sp-d16
    -mfloat-abi=hard
)

add_compile_options(
    ${CPU_FLAGS}
    -Wall
    -Wextra
    -fdata-sections
    -ffunction-sections
    -g3
    -O0
)

# 4. 全局宏定义（告知底层库芯片具体型号）
add_compile_definitions(
    STM32F405xx
    USE_HAL_DRIVER
)

# 5. 头文件搜索路径
set(INC_DIRS
    ${CMAKE_CURRENT_SOURCE_DIR}/Core/Inc
    ${CMAKE_CURRENT_SOURCE_DIR}/Drivers/CMSIS/Include
    ${CMAKE_CURRENT_SOURCE_DIR}/Drivers/CMSIS/Device/ST/STM32F4xx/Include
    ${CMAKE_CURRENT_SOURCE_DIR}/Drivers/STM32F4xx_HAL_Driver/Inc
    ${CMAKE_CURRENT_SOURCE_DIR}/Drivers/STM32F4xx_HAL_Driver/Inc/Legacy
)

# 6. 源文件列表
# 注意：启动文件、系统初始化与用户代码
set(SRCS
    Core/Src/main.c
    Core/Src/system_stm32f4xx.c
    Core/Src/stm32f4xx_it.c
    Core/Src/startup_stm32f405xx.s

    # HAL 基础驱动（可按需增减）
    Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal.c
    Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_cortex.c
    Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_rcc.c
    Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_rcc_ex.c    # 若用到 PLL 时钟建议一并加入
    Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_flash.c     # 👈 必须添加
    Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_flash_ex.c  # 👈 必须添加
    Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_gpio.c
)

# 7. 创建构建目标 .elf
add_executable(${PROJECT_NAME}.elf ${SRCS})

target_include_directories(${PROJECT_NAME}.elf PRIVATE ${INC_DIRS})

# 8. 链接参数配置
set(LINKER_SCRIPT ${CMAKE_CURRENT_SOURCE_DIR}/STM32F405RGTx_FLASH.ld)

target_link_options(${PROJECT_NAME}.elf PRIVATE
    ${CPU_FLAGS}
    -T${LINKER_SCRIPT}
    -specs=nano.specs
    -specs=nosys.specs
    -Wl,-Map=${PROJECT_BINARY_DIR}/${PROJECT_NAME}.map,--cref
    -Wl,--gc-sections
)

# 9. 编译后自动生成 .bin、.hex 并打印固件体积
add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${PROJECT_NAME}.elf> ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.bin
    COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${PROJECT_NAME}.elf> ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.hex
    COMMAND ${CMAKE_SIZE_UTIL} --format=berkeley $<TARGET_FILE:${PROJECT_NAME}.elf>
    COMMENT "Generating .bin, .hex and printing firmware size"
)