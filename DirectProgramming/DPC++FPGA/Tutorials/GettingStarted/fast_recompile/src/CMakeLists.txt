set(DEVICE_SOURCE_FILE kernel.cpp)
set(KERNEL_HEADER_FILE kernel.hpp)
set(HOST_SOURCE_FILE main.cpp)
set(TARGET_NAME fast_recompile)
set(EMULATOR_TARGET ${TARGET_NAME}.fpga_emu)
set(FPGA_TARGET ${TARGET_NAME}.fpga)


# FPGA board selection
set(A10_PAC_BOARD_NAME "intel_a10gx_pac:pac_a10")
set(S10_PAC_BOARD_NAME "intel_s10sx_pac:pac_s10")
set(SELECTED_BOARD ${A10_PAC_BOARD_NAME})
if (NOT DEFINED FPGA_BOARD)
    message(STATUS "\tFPGA_BOARD was not specified. Configuring the design to run on the Intel(R) Programmable Acceleration Card (PAC) with Intel Arria(R) 10 GX FPGA. Please refer to the README for information on board selection.")
elseif(FPGA_BOARD STREQUAL ${A10_PAC_BOARD_NAME})
    message(STATUS "\tConfiguring the design to run on the Intel(R) Programmable Acceleration Card (PAC) with Intel Arria(R) 10 GX FPGA.")
elseif(FPGA_BOARD STREQUAL ${S10_PAC_BOARD_NAME})
    message(STATUS "\tConfiguring the design to run on the Intel(R) Programmable Acceleration Card (PAC) D5005 (with Intel Stratix(R) 10 SX FPGA).")
    set(SELECTED_BOARD ${S10_PAC_BOARD_NAME})
else()
    message(STATUS "\tAn invalid board name was passed in using the FPGA_BOARD flag. Configuring the design to run on the Intel(R) Programmable Acceleration Card (PAC) with Intel Arria(R) 10 GX FPGA. Please refer to the README for the list of valid board names.")
endif()


# Flags
set(EMULATOR_COMPILE_FLAGS -fintelfpga -DFPGA_EMULATOR -c)
set(EMULATOR_LINK_FLAGS -fintelfpga)
set(HARDWARE_COMPILE_FLAGS -fintelfpga -c)
set(HARDWARE_LINK_FLAGS "-fintelfpga -Xshardware -Xsboard=${SELECTED_BOARD} ${USER_HARDWARE_FLAGS}")
# use cmake -D USER_HARDWARE_FLAGS=<flags> to set extra flags for FPGA backend compilation

String(STRIP "${CMAKE_EXE_LINKER_FLAGS}" CMAKE_EXE_LINKER_FLAGS)


# FPGA emulator
if(WIN32)
    set(WIN_EMULATOR_TARGET ${EMULATOR_TARGET}.exe)
    add_custom_target(fpga_emu DEPENDS ${WIN_EMULATOR_TARGET})
    set(HOST_EMU_OBJ "host_emu.o")
    set(DEVICE_EMU_OBJ "dev_emu.o")
    set(DEVICE_IMAGE_EMU_OBJ "dev_image_emu.a")

    add_custom_command(OUTPUT ${HOST_EMU_OBJ} 
                 COMMAND dpcpp /EHsc ${EMULATOR_COMPILE_FLAGS}
                 ${CMAKE_CURRENT_SOURCE_DIR}/${HOST_SOURCE_FILE} -o ${HOST_EMU_OBJ} 
                 DEPENDS ${HOST_SOURCE_FILE} ${KERNEL_HEADER_FILE})

    add_custom_command(OUTPUT ${DEVICE_EMU_OBJ} 
                 COMMAND dpcpp /EHsc ${EMULATOR_COMPILE_FLAGS} ${CMAKE_CURRENT_SOURCE_DIR}/${DEVICE_SOURCE_FILE} -o ${DEVICE_EMU_OBJ}
                 DEPENDS ${DEVICE_SOURCE_FILE} ${KERNEL_HEADER_FILE})

    add_custom_command(OUTPUT ${DEVICE_IMAGE_EMU_OBJ} 
                 COMMAND dpcpp ${EMULATOR_LINK_FLAGS} -fsycl-link=image ${DEVICE_EMU_OBJ} -o ${DEVICE_IMAGE_EMU_OBJ} 
                 DEPENDS ${DEVICE_EMU_OBJ})

    add_custom_command(OUTPUT ${WIN_EMULATOR_TARGET}
                 COMMAND dpcpp -fintelfpga ${HOST_EMU_OBJ} ${DEVICE_IMAGE_EMU_OBJ} -o ${CMAKE_BINARY_DIR}/${WIN_EMULATOR_TARGET} 
                 DEPENDS ${HOST_EMU_OBJ} ${DEVICE_IMAGE_EMU_OBJ})
                 
else()
    add_custom_target(fpga_emu DEPENDS ${EMULATOR_TARGET})
    set(HOST_EMU_OBJ "host_emu.o")
    set(DEVICE_EMU_OBJ "dev_emu.o")
    set(DEVICE_IMAGE_EMU_OBJ "dev_image_emu.a")

    separate_arguments(CMAKE_CXX_FLAGS_LIST UNIX_COMMAND "${CMAKE_CXX_FLAGS}")

    add_custom_command(OUTPUT ${HOST_EMU_OBJ} 
                 COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS_LIST} ${EMULATOR_COMPILE_FLAGS}
                 ${CMAKE_CURRENT_SOURCE_DIR}/${HOST_SOURCE_FILE} -o ${HOST_EMU_OBJ} 
                 DEPENDS ${HOST_SOURCE_FILE} ${KERNEL_HEADER_FILE})

    add_custom_command(OUTPUT ${DEVICE_EMU_OBJ} 
                 COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS_LIST} ${EMULATOR_COMPILE_FLAGS} ${CMAKE_CURRENT_SOURCE_DIR}/${DEVICE_SOURCE_FILE} -o ${DEVICE_EMU_OBJ}
                 DEPENDS ${DEVICE_SOURCE_FILE} ${KERNEL_HEADER_FILE})

    add_custom_command(OUTPUT ${DEVICE_IMAGE_EMU_OBJ} 
                 COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS_LIST} ${EMULATOR_LINK_FLAGS} -fsycl-link=image ${DEVICE_EMU_OBJ} -o ${DEVICE_IMAGE_EMU_OBJ} 
                 DEPENDS ${DEVICE_EMU_OBJ})

    add_custom_command(OUTPUT ${EMULATOR_TARGET}
                 COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS_LIST} -fintelfpga ${HOST_EMU_OBJ} ${DEVICE_IMAGE_EMU_OBJ} -o ${CMAKE_BINARY_DIR}/${EMULATOR_TARGET} 
                 DEPENDS ${HOST_EMU_OBJ} ${DEVICE_IMAGE_EMU_OBJ})
endif()

# FPGA hardware
if(WIN32)
    add_custom_target(fpga COMMAND echo "An FPGA hardware target is not provided on Windows. See README for details.")
else()
    add_custom_target(fpga DEPENDS ${FPGA_TARGET})
    set(HOST_OBJ "host.o")
    set(DEVICE_OBJ "dev.o")
    set(DEVICE_IMAGE_OBJ "dev_image.a")

    separate_arguments(CMAKE_CXX_FLAGS_LIST UNIX_COMMAND "${CMAKE_CXX_FLAGS}")
    separate_arguments(HARDWARE_LINK_FLAGS_LIST UNIX_COMMAND "${HARDWARE_LINK_FLAGS}")

    add_custom_command(OUTPUT ${HOST_OBJ} 
                 COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS_LIST} ${HARDWARE_COMPILE_FLAGS} ${CMAKE_CURRENT_SOURCE_DIR}/${HOST_SOURCE_FILE} -o ${HOST_OBJ} 
                 DEPENDS ${HOST_SOURCE_FILE} ${KERNEL_HEADER_FILE})

    add_custom_command(OUTPUT ${DEVICE_OBJ} 
                 COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS_LIST} ${HARDWARE_COMPILE_FLAGS} ${CMAKE_CURRENT_SOURCE_DIR}/${DEVICE_SOURCE_FILE} -o ${DEVICE_OBJ} 
                 DEPENDS ${DEVICE_SOURCE_FILE} ${KERNEL_HEADER_FILE})

    add_custom_command(OUTPUT ${DEVICE_IMAGE_OBJ} 
                 COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS_LIST} ${HARDWARE_LINK_FLAGS_LIST} -fsycl-link=image ${DEVICE_OBJ} -o ${DEVICE_IMAGE_OBJ} 
                 DEPENDS ${DEVICE_OBJ})

    add_custom_command(OUTPUT ${FPGA_TARGET} 
                 COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS_LIST} ${CMAKE_EXE_LINKER_FLAGS} -fintelfpga ${HOST_OBJ} ${DEVICE_IMAGE_OBJ} -o ${CMAKE_BINARY_DIR}/${FPGA_TARGET} 
                 DEPENDS ${HOST_OBJ} ${DEVICE_IMAGE_OBJ})
endif()

