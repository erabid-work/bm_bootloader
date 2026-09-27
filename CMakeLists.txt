cmake_minimum_required(VERSION 3.20)
project(baremetal_bootloader C ASM)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

set(BOOTLOADER_START "0x08000000" CACHE STRING "Bootloader flash start")
set(BOOTLOADER_SIZE  "0x00008000" CACHE STRING "Bootloader flash size")
set(BANK_A_START     "0x08008000" CACHE STRING "Application bank A start")
set(BANK_A_SIZE      "0x00038000" CACHE STRING "Application bank A size")
set(BANK_B_START     "0x08040000" CACHE STRING "Application bank B start")
set(BANK_B_SIZE      "0x00038000" CACHE STRING "Application bank B size")
set(METADATA_START   "0x08078000" CACHE STRING "Metadata address")
set(METADATA_SIZE    "0x00008000" CACHE STRING "Metadata size")

set(BOOT_TRANSPORT "NONE" CACHE STRING "NONE/UART/USB/ETHERNET")
set(BOOT_VALIDATION "CRC32" CACHE STRING "NONE/CRC32")
option(BOOT_ENABLE_ROLLBACK "Enable A/B rollback" ON)
option(BOOT_ENABLE_UPDATE "Enable firmware update" ON)

set(SOURCES
    src/main.c
    src/config/boot_config.c
    src/bank/bank_manager.c
    src/image/image_manager.c
    src/validation/image_validation.c
    src/metadata/boot_metadata.c
    src/transport/firmware_transport.c
    src/jump/application_jump.c
    src/platform/platform.c
    src/platform/platform_flash.c
    port/cortex_m/startup.c
)

add_executable(${PROJECT_NAME}.elf ${SOURCES})

target_include_directories(${PROJECT_NAME}.elf PRIVATE include src)

target_compile_definitions(${PROJECT_NAME}.elf PRIVATE
    BOOTLOADER_START=${BOOTLOADER_START}
    BOOTLOADER_SIZE=${BOOTLOADER_SIZE}
    BANK_A_START=${BANK_A_START}
    BANK_A_SIZE=${BANK_A_SIZE}
    BANK_B_START=${BANK_B_START}
    BANK_B_SIZE=${BANK_B_SIZE}
    METADATA_START=${METADATA_START}
    METADATA_SIZE=${METADATA_SIZE}
    BOOT_TRANSPORT_${BOOT_TRANSPORT}
    BOOT_VALIDATION_${BOOT_VALIDATION}
    BOOT_ENABLE_ROLLBACK=$<BOOL:${BOOT_ENABLE_ROLLBACK}>
    BOOT_ENABLE_UPDATE=$<BOOL:${BOOT_ENABLE_UPDATE}>
)

target_compile_options(${PROJECT_NAME}.elf PRIVATE
    -mcpu=cortex-m4 -mthumb
    -ffunction-sections -fdata-sections
    -Wall -Wextra -Werror=return-type
)

target_link_options(${PROJECT_NAME}.elf PRIVATE
    -mcpu=cortex-m4 -mthumb
    -T${CMAKE_CURRENT_SOURCE_DIR}/port/cortex_m/linker.ld
    -Wl,--gc-sections
    -Wl,-Map=${PROJECT_NAME}.map
    --specs=nosys.specs --specs=nano.specs
)

add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${PROJECT_NAME}.elf>
            ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.hex
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${PROJECT_NAME}.elf>
            ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.bin
)
