set(SOURCE_FILE use_library.cpp)
set(HEADER_FILE lib.hpp)
set(LIBRARY_ARCHIVE lib.a)
set(TARGET_NAME use_library)
set(EMULATOR_TARGET ${TARGET_NAME}.fpga_emu)
set(FPGA_TARGET ${TARGET_NAME}.fpga)
set(REPORT_TARGET ${TARGET_NAME}_report.a)

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

# Library source files
set(HLS_SOURCE lib_hls.cpp) 
set(HLS_SOURCE_OBJECT lib_hls.o)
set(OCL_SOURCE lib_ocl.cl) 
set(OCL_SOURCE_OBJECT lib_ocl.o)
set(SYCL_SOURCE lib_sycl.cpp) 
set(SYCL_SOURCE_OBJECT lib_sycl.o)
set(RTL_C_MODEL lib_rtl_model.cpp)
set(RTL_SPEC lib_rtl_spec.xml) 
set(RTL_V lib_rtl.v) 
set(RTL_SOURCE_OBJECT lib_rtl.o)

# Flags
set(LIBRARY_DEVICE_LINK_FLAGS "${LIBRARY_ARCHIVE}")
set(LIBRARY_HOST_LINK_FLAGS "${HLS_SOURCE_OBJECT} ${OCL_SOURCE_OBJECT} ${SYCL_SOURCE_OBJECT} ${RTL_SOURCE_OBJECT}")
set(EMULATOR_COMPILE_FLAGS "-fintelfpga -DFPGA_EMULATOR")
set(EMULATOR_LINK_FLAGS "-fintelfpga ${LIBRARY_DEVICE_LINK_FLAGS}")
set(HARDWARE_COMPILE_FLAGS "-fintelfpga")
set(HARDWARE_LINK_FLAGS "-fintelfpga -Xshardware -Xsboard=${SELECTED_BOARD} ${LIBRARY_DEVICE_LINK_FLAGS} ${USER_HARDWARE_FLAGS}")
# use cmake -D USER_HARDWARE_FLAGS=<flags> to set extra flags for FPGA backend compilation

# Flag lists for custom commands
separate_arguments(CMAKE_CXX_FLAGS_LIST UNIX_COMMAND "${CMAKE_CXX_FLAGS}")
separate_arguments(EMULATOR_COMPILE_FLAGS_LIST UNIX_COMMAND "${EMULATOR_COMPILE_FLAGS}")
separate_arguments(EMULATOR_LINK_FLAGS_LIST UNIX_COMMAND "${EMULATOR_LINK_FLAGS}")
separate_arguments(HARDWARE_COMPILE_FLAGS_LIST UNIX_COMMAND "${HARDWARE_COMPILE_FLAGS}")
separate_arguments(HARDWARE_LINK_FLAGS_LIST UNIX_COMMAND "${HARDWARE_LINK_FLAGS}")

# Create HLS source object
add_custom_target(
    create_hls_source_object 
    COMMAND fpga_crossgen ${HLS_SOURCE} --source hls --target sycl -o ${HLS_SOURCE_OBJECT} ${CMAKE_CXX_FLAGS_LIST}
)

# Create OCL source object
add_custom_target(
    create_ocl_source_object
    COMMAND fpga_crossgen ${OCL_SOURCE} --source ocl --target sycl -o ${OCL_SOURCE_OBJECT}
)

# Create SYCL source object
add_custom_target(
    create_sycl_source_object 
    COMMAND fpga_crossgen ${SYCL_SOURCE} --source sycl --target sycl -o ${SYCL_SOURCE_OBJECT} ${CMAKE_CXX_FLAGS_LIST}
)

# Create RTL source object
add_custom_target(
    create_rtl_source_object
    COMMAND fpga_crossgen ${RTL_SPEC} --emulation_model ${RTL_C_MODEL} --target sycl -o ${RTL_SOURCE_OBJECT}
)

# Create library archive
add_custom_target(
    create_library_archive 
    COMMAND fpga_libtool ${HLS_SOURCE_OBJECT} ${OCL_SOURCE_OBJECT} ${SYCL_SOURCE_OBJECT} ${RTL_SOURCE_OBJECT} --target sycl --create ${LIBRARY_ARCHIVE} 
    DEPENDS create_hls_source_object create_ocl_source_object create_sycl_source_object create_rtl_source_object
)

# FPGA emulator
set(SOURCE_OBJ_FILE_EMU ${SOURCE_FILE}.emu.o)
add_custom_target(fpga_emu DEPENDS ${EMULATOR_TARGET})
add_custom_command(OUTPUT ${SOURCE_OBJ_FILE_EMU}
                   COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS_LIST} ${SOURCE_FILE} ${EMULATOR_COMPILE_FLAGS_LIST} -c -o ${SOURCE_OBJ_FILE_EMU}
                   DEPENDS ${SOURCE_FILE} ${HEADER_FILE})
add_custom_command(OUTPUT ${EMULATOR_TARGET}
                   COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS_LIST} ${SOURCE_OBJ_FILE_EMU} ${EMULATOR_LINK_FLAGS_LIST} -o ${CMAKE_BINARY_DIR}/${EMULATOR_TARGET}
                   DEPENDS ${SOURCE_OBJ_FILE_EMU} create_library_archive)

# FPGA hardware
set(SOURCE_OBJ_FILE_FPGA ${SOURCE_FILE}.fpga.o)
add_custom_target(fpga DEPENDS ${FPGA_TARGET})
add_custom_command(OUTPUT ${SOURCE_OBJ_FILE_FPGA}
                   COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS_LIST} ${SOURCE_FILE} ${HARDWARE_COMPILE_FLAGS_LIST} -c -o ${SOURCE_OBJ_FILE_FPGA}
                   DEPENDS ${SOURCE_FILE} ${HEADER_FILE})
add_custom_command(OUTPUT ${FPGA_TARGET}
                   COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS_LIST} ${SOURCE_OBJ_FILE_FPGA} ${HARDWARE_LINK_FLAGS_LIST} -o ${CMAKE_BINARY_DIR}/${FPGA_TARGET}
                   DEPENDS ${SOURCE_OBJ_FILE_FPGA} create_library_archive)

# Generate report
set(SOURCE_OBJ_FILE_REPORT ${SOURCE_FILE}.report.o)
add_custom_target(report DEPENDS ${REPORT_TARGET})
add_custom_command(OUTPUT ${SOURCE_OBJ_FILE_REPORT}
                   COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS_LIST} ${SOURCE_FILE} ${HARDWARE_COMPILE_FLAGS_LIST} -c -o ${SOURCE_OBJ_FILE_REPORT}
                   DEPENDS ${SOURCE_FILE} ${HEADER_FILE})
add_custom_command(OUTPUT ${REPORT_TARGET} 
                   COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS_LIST} ${SOURCE_OBJ_FILE_REPORT} ${HARDWARE_LINK_FLAGS_LIST} -fsycl-link -o ${CMAKE_BINARY_DIR}/${REPORT_TARGET}
                   DEPENDS ${SOURCE_OBJ_FILE_REPORT} create_library_archive)

# Copy files
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${SOURCE_FILE} ${SOURCE_FILE} COPYONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${HEADER_FILE} ${HEADER_FILE} COPYONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${HLS_SOURCE} ${HLS_SOURCE} COPYONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${OCL_SOURCE} ${OCL_SOURCE} COPYONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${SYCL_SOURCE} ${SYCL_SOURCE} COPYONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${RTL_SPEC} ${RTL_SPEC} COPYONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${RTL_C_MODEL} ${RTL_C_MODEL} COPYONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${RTL_V} ${RTL_V} COPYONLY)

