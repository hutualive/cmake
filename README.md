# cmake
use cmake as build system

# project tree
basePrj/  
|------ CMakeLists.txt  
|------ libs  
|       *------  hello  
│       *------  CMakeLists.txt  
│       *------  inc  
|               **------ hello.h  
│       *------  src  
│               **------ hello.c  
|------ src  
        *------ main.c  

// top-level CMake file  
cmake_minimum_required(VERSION 3.16.4)  

project(basePrj)  

add_subdirectory(libs/hello)  

set(SOURCES src/main.c)  

add_executable(${PROJECT_NAME} ${SOURCES})  
target_link_libraries(${PROJECT_NAME} hello)  

# build process
$ cd basePrj  
$ mkdir build  
$ cd build  
$ cmake ..  
$ make  
$ ./basePrj  
Hello World!  

# setup CMake
// cross compiling for arm
set(CMAKE_SYSTEM_NAME Generic)  
set(CMAKE_SYSTEM_PROCESSOR arm)  
set(CMAKE_CROSSCOMPILING 1)  

// toolchain
set(GCC_PATH ${TOOLCHAIN_PATH}/gcc-arm-none-eabi-9-2019-q4-major/bin)  
set(CMAKE_C_COMPILER ${GCC_PATH}/arm-none-eabi-gcc CACHE PATH "" FORCE)  

// compiler flags for target
set(CMAKE_C_FLAGS "-mcpu=cortex-m4 -march=armv7e-m -mthumb -mfloat-abi=hard
-mfpu=fpv4-sp-d16 -DPART_STM32F411VExx -ffunction-sections -fdata-sections
-Wall -specs=\"nosys.specs\"")  

// compiler flags for debug or release build
set(CMAKE_C_FLAGS_RELEASE "-Os")  
set(CMAKE_C_FLAGS_DEBUG "-Og -g -gdwarf-3 -gstrict-dwarf")  

// linker flags for target
set(CMAKE_EXE_LINKER_FLAGS "--entry Reset_Handler -Wl,-T\"../stm32f411vexx.ld\"")  

// objcopy with cmake  --> ALL specify the target is run automatically with make  
set(CMAKE_C_OBJCOPY ${GCC_PATH}/arm-none-eabi-objcopy CACHE PATH "" FORCE)  

add_custom_target(${PROJECT_NAME}.bin ALL DEPENDS ${PROJECT_NAME}.elf)  
add_custom_command(TARGET ${PROJECT_NAME}.bin  
    COMMAND ${CMAKE_C_OBJCOPY} ARGS -O binary ${PROJECT_NAME}.elf  
    ${PROJECT_NAME}.bin)  

// flash with cmake  --> to be fixed with pyocd or openocd  
// without ALL  --> does not run automatically  
// USES_TERMINAL  --> see the output from JLinkExe
add_custom_target(flash DEPENDS ${PROJECT_NAME}.bin)
add_custom_command(TARGET flash
    USES_TERMINAL
    COMMAND JLinkExe -CommanderScript ../flash.jlink)

// debugging  --> create a debug directory
$ cd targetPrj  
$ mkdir debug  
$ cd debug  
$ cmake -DCMAKE_BUILD_TYPE=Debug ..  
$ make  

// launch gdb server  
$ JLinkGDBServer -if JTAG -device STM32F411VExx

// launch gdb
$ arm-none-eabi-gdb targetPrj.elf  
(gdb) target remote :2331  
mon reset  
load  

// wrap to a cmake command  --> using tmux  
set(CMAKE_C_GDB ${GCC_PATH}/arm-none-eabi-gdb CACHE PATH "" FORCE)  

add_custom_target(gdb DEPENDS ${PROJECT_NAME}.elf)  
add_custom_command(TARGET gdb  
    COMMAND tmux -e JLinkGDBServer -if JTAG -device STM32F411VExx &  
    COMMAND tmux -e ${CMAKE_C_GDB} ${PROJECT_NAME}.elf -x ../gdbInit &)  

