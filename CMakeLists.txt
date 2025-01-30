# Derived from the Pico SDK, which carries the following
# LICENSE.txt:
# Copyright 2020 (c) 2020 Raspberry Pi (Trading) Ltd.
#
# Redistribution and use in source and binary forms, with or without modification, are permitted provided that the
# following conditions are met:
#
# 1. Redistributions of source code must retain the above copyright notice, this list of conditions and the following
#    disclaimer.
#
# 2. Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following
#    disclaimer in the documentation and/or other materials provided with the distribution.
#
# 3. Neither the name of the copyright holder nor the names of its contributors may be used to endorse or promote products
#    derived from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES,
# INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
# DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
# SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
# SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
# WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF
# THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

cmake_minimum_required(VERSION 3.13)

include(pico_sdk_import.cmake)

project(test_project C CXX ASM)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)

pico_sdk_init()

# Build the bootloader as a standalone thing

add_executable(bootloader main.c)

function(target_cl_options option)
    target_compile_options(bootloader PRIVATE ${option})
    target_link_options(bootloader PRIVATE ${option})
endfunction()

target_cl_options("-Os")
target_cl_options("-ffunction-sections")
target_cl_options("-fdata-sections")
target_link_options(bootloader PRIVATE "LINKER:--gc-sections")

pico_add_extra_outputs(bootloader)
pico_set_binary_type(bootloader copy_to_ram)

set_target_properties(bootloader PROPERTIES COMPILE_FLAGS "-Wall")

pico_set_linker_script(bootloader ${CMAKE_CURRENT_SOURCE_DIR}/bootloader.ld)
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
	target_compile_definitions(bootloader PRIVATE DEBUG)
	pico_enable_stdio_usb(bootloader 1)
endif()
if(${SKIP_BOOTLOADER_ENTRY_PIN})
	message(STATUS "Skipping bootloader entry pin")
	add_definitions(-DSKIP_BOOTLOADER_ENTRY_PIN)
endif()
target_link_libraries(bootloader
                      pico_stdlib
                      hardware_dma
                      hardware_flash
                      hardware_structs
                      hardware_resets

                      cmsis_core)

set(BOOTLOADER_DIR "${CMAKE_CURRENT_LIST_DIR}" CACHE INTERNAL "")

# Build a library to embed into applications

function(bootloader_define_library)
	set(NAME bootloader)
	set(ORIGINAL_BIN ${CMAKE_CURRENT_BINARY_DIR}/${NAME}.bin)
	set(BIN_ASM ${CMAKE_CURRENT_BINARY_DIR}/${NAME}_bin.S)

	add_custom_target(${NAME}_bin DEPENDS ${ORIGINAL_BIN})
	add_custom_command(OUTPUT ${ORIGINAL_BIN} DEPENDS ${NAME} COMMAND ${CMAKE_OBJCOPY} -Obinary $<TARGET_FILE:${NAME}> ${ORIGINAL_BIN})

	find_package (Python3 REQUIRED COMPONENTS Interpreter)
	add_custom_target(${NAME}_bin_asm DEPENDS ${BIN_ASM})
	add_custom_command(OUTPUT ${BIN_ASM} DEPENDS ${ORIGINAL_BIN}
		COMMAND ${Python3_EXECUTABLE} ${BOOTLOADER_DIR}/mkasm.py ${ORIGINAL_BIN} ${BIN_ASM}
	)

	add_library(${NAME}_library INTERFACE)
	add_dependencies(${NAME}_library ${NAME}_bin_asm)
	# not strictly (or indeed actually) a link library, but this avoids dependency cycle
	target_link_libraries(${NAME}_library INTERFACE ${BIN_ASM})
endfunction()

bootloader_define_library()

# Provide a helper to build a combined target

function(bootloader_build_combined NAME)
	set(APP ${NAME}_app)
	set(APP_BIN ${CMAKE_CURRENT_BINARY_DIR}/${APP}.bin)
	set(APP_HDR ${CMAKE_CURRENT_BINARY_DIR}/${APP}_hdr.bin)

	set(COMBINED ${NAME}_combined)

	target_link_libraries(${NAME} bootloader_library)
	if(CMAKE_BUILD_TYPE STREQUAL "Debug")
		pico_set_linker_script(${NAME} ${BOOTLOADER_DIR}/combined_debug.ld)
		set(SKIP 44)
	else()
		pico_set_linker_script(${NAME} ${BOOTLOADER_DIR}/combined.ld)
		set(SKIP 16)
	endif()
	MATH(EXPR IMAGE_HDR 0x10000000+${SKIP}*1024)
	message(STATUS "IMAGE_HDR: ${IMAGE_HDR}")
	

	pico_add_bin_output(${NAME})

	add_custom_target(${APP}_bin DEPENDS ${APP_BIN} ${NAME})
	add_custom_command(OUTPUT ${APP_BIN} DEPENDS ${NAME}.bin
		COMMAND dd ibs=1k skip=${SKIP} if=${NAME}.bin of=${APP_BIN}
	)
	add_custom_target(${APP}_hdr DEPENDS ${APP}_bin ${NAME})
	add_custom_command(OUTPUT ${APP_HDR} DEPENDS ${APP_BIN} ${NAME}
		COMMAND ${BOOTLOADER_DIR}/gen_imghdr.py -a ${IMAGE_HDR} ${APP_BIN} ${APP_HDR}
	)

	add_custom_target(${COMBINED} ALL DEPENDS ${APP_HDR} ${NAME})
	add_custom_command(TARGET ${COMBINED} DEPENDS ${APP_HDR} ${NAME}
		COMMAND ${CMAKE_OBJCOPY} --update-section .app_hdr=${APP_HDR} ${NAME}.elf ${COMBINED}.elf
	)
	add_custom_command(TARGET ${COMBINED} POST_BUILD
		COMMAND ${CMAKE_OBJCOPY} -Obinary ${COMBINED}.elf ${COMBINED}.bin
	)
	add_custom_command(TARGET ${COMBINED} POST_BUILD # This uf2 doesn't work as it has the correct ld script, but the rest wrong.
		COMMAND rm ${NAME}.uf2 > /dev/null 2>&1  || true
	)
	if(PICO_PLATFORM STREQUAL "rp2040")
		set(UF2_FAMILY "0xe48bff56")
	elseif(PICO_PLATFORM STREQUAL "rp2350-arm-s")
		set(UF2_FAMILY "0xe48bff59")
	elseif(PICO_PLATFORM STREQUAL "rp2350-riscv")
		set(UF2_FAMILY "0xe48bff5a")
	else()
		message(FATAL_ERROR "Unknown PICO_PLATFORM: ${PICO_PLATFORM}")
	endif()
	# Assumes picotool is installed to convert the elf to uf2.
	# Needs to know the family type to generate the correct uf2.
	add_custom_command(TARGET ${COMBINED} POST_BUILD
		COMMAND picotool uf2 convert ${COMBINED}.elf ${COMBINED}.uf2 --family ${UF2_FAMILY})
endfunction()

# Provide a helper to build a standalone target

function(bootloader_build_standalone NAME)
	pico_set_linker_script(${NAME} ${BOOTLOADER_DIR}/standalone.ld)
	pico_add_bin_output(${NAME})
endfunction()
