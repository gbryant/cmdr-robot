cmake_minimum_required(VERSION 3.20)

set(PICO_BOARD pico2_w CACHE STRING "Board type")
include($ENV{PICO_SDK_PATH}/external/pico_sdk_import.cmake)
include(${CMAKE_CURRENT_LIST_DIR}/FreeRTOS_Kernel_import.cmake)

project(cmdr-robot C CXX ASM)
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
add_compile_definitions(CYW43_ENABLE_BLUETOOTH=1)  # commander controller (Bluetooth)
pico_sdk_init()

include(FetchContent)
FetchContent_Declare(commander
    GIT_REPOSITORY https://github.com/gbryant/commander.git
    GIT_TAG        v1.1     # release pin — move with `cmdr pin <tag>`
)
set(COMMANDER_ENABLE_CONTROLLER ON CACHE BOOL "" FORCE)  # commander controller
set(COMMANDER_ENABLE_OTA ON CACHE BOOL "" FORCE)
add_compile_definitions(MAX_COMMANDS=24)  # was 16 (full → silently dropped `ota`, which
                                          # registers last). 17 in use now; headroom for app cmds.
FetchContent_MakeAvailable(commander)

if(DEFINED ENV{PFB_PATH})
    set(PFB_PATH "$ENV{PFB_PATH}")
else()
    set(PFB_PATH "$ENV{HOME}/u-developer/pico_fota_bootloader")
endif()
set(PFB_WITH_IMAGE_ENCRYPTION OFF CACHE BOOL "" FORCE)
set(PFB_WITH_SHA256_HASHING   ON  CACHE BOOL "" FORCE)
add_subdirectory(${PFB_PATH} pfb_build)

add_executable(cmdr-robot main.cpp)
commander_generate_scripts(cmdr-robot)
target_link_libraries(cmdr-robot PRIVATE commander::pico_runner pico_fota_bootloader_lib)
pico_enable_stdio_usb(cmdr-robot 1)
pico_enable_stdio_uart(cmdr-robot 0)
pico_add_extra_outputs(cmdr-robot)
pfb_compile_with_bootloader(cmdr-robot)
