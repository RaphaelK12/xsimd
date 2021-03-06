############################################################################
# Copyright (c) Johan Mabille, Sylvain Corlay, Wolf Vollprecht and         #
# Martin Renou                                                             #
# Copyright (c) QuantStack                                                 #
#                                                                          #
# Distributed under the terms of the BSD 3-Clause License.                 #
#                                                                          #
# The full license is in the file LICENSE, distributed with this software. #
############################################################################

cmake_minimum_required(VERSION 3.1)

if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    project(xsimd-test)

    find_package(xsimd REQUIRED CONFIG)
    set(XSIMD_INCLUDE_DIR ${xsimd_INCLUDE_DIRS})
endif ()

message(STATUS "Forcing tests build type to Release")
set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)

include(CheckCXXCompilerFlag)

string(TOUPPER "${CMAKE_BUILD_TYPE}" U_CMAKE_BUILD_TYPE)

################
# ARM SETTINGS #
################

OPTION(CROSS_COMPILE_ARM "cross compile for ARM targets" OFF)

# Note: to compile on ARM (or cross compile), you may need to add the following:
# -DTARGET_ARCH="armv8-a -mfpu=neon -mfloat-abi=softfp -target arm-linux-gnueabi"
set(TARGET_ARCH "native" CACHE STRING "Target architecture arguments")
set(ARM_ARCH_DIRECTORY "arm-linux-gnueabi" CACHE STRING "ARM arch header dir")
set(ARM_GCC_VER "4.7.3" CACHE STRING "ARM GCC header dir")

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID MATCHES "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES "Intel")
    if (CROSS_COMPILE_ARM)
        # We're cross-compiling with clang++ on Travis, this is all pretty specific and just for testing
        set(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS)
        set(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS)
        set(CMAKE_THREAD_LIBS_INIT)

        include_directories(/usr/${ARM_ARCH_DIRECTORY}/include/c++/${ARM_GCC_VER}/${ARM_ARCH_DIRECTORY}/)
        include_directories(/usr/${ARM_ARCH_DIRECTORY}/include/c++/${ARM_GCC_VER}/)
        include_directories(/usr/${ARM_ARCH_DIRECTORY}/include/)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=${TARGET_ARCH} -Wunused-parameter -Wextra -Wreorder -std=c++11")
    elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "^ppc64")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -fPIC -mcpu=${TARGET_ARCH} -mtune=${TARGET_ARCH} -Wunused-parameter -Wextra -Wreorder -std=c++11")
    elseif(NOT WIN32)
        CHECK_CXX_COMPILER_FLAG("-std=c++11" HAS_CPP11_FLAG)
        if (ENABLE_XTL_COMPLEX)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -fPIC -march=${TARGET_ARCH} -Wunused-parameter -Wextra -Wreorder -std=c++14")
        elseif (HAS_CPP11_FLAG)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -fPIC -march=${TARGET_ARCH} -Wunused-parameter -Wextra -Wreorder -std=c++11")
        else()
            message(FATAL_ERROR "Unsupported compiler -- xsimd requires C++11 support!")
        endif()
    endif()
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHsc /MP /bigobj")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /arch:AVX2")
    set(CMAKE_EXE_LINKER_FLAGS /MANIFEST:NO)
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES Clang AND WIN32) # We are using clang-cl
    add_compile_options(/EHsc /bigobj)
    set(CMAKE_EXE_LINKER_FLAGS /MANIFEST:NO)
endif()

if(DOWNLOAD_GTEST OR GTEST_SRC_DIR)
    if(DOWNLOAD_GTEST)
        # Download and unpack googletest at configure time
        configure_file(downloadGTest.cmake.in googletest-download/CMakeLists.txt)
    else()
        # Copy local source of googletest at configure time
        configure_file(copyGTest.cmake.in googletest-download/CMakeLists.txt)
    endif()
    execute_process(COMMAND ${CMAKE_COMMAND} -G "${CMAKE_GENERATOR}" .
                    RESULT_VARIABLE result
                    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/googletest-download )
    if(result)
        message(FATAL_ERROR "CMake step for googletest failed: ${result}")
    endif()
    execute_process(COMMAND ${CMAKE_COMMAND} --build .
                    RESULT_VARIABLE result
                    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/googletest-download )
    if(result)
        message(FATAL_ERROR "Build step for googletest failed: ${result}")
    endif()

    set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
    
    # Add googletest directly to our build. This defines
    # the gtest and gtest_main targets.
    add_subdirectory(${CMAKE_CURRENT_BINARY_DIR}/googletest-src
                     ${CMAKE_CURRENT_BINARY_DIR}/googletest-build EXCLUDE_FROM_ALL)

    set(GTEST_INCLUDE_DIRS "${gtest_SOURCE_DIR}/include")
    set(GTEST_BOTH_LIBRARIES  gtest_main gtest)
else()
    find_package(GTest REQUIRED)
endif()

find_package(Threads)

include_directories(${GTEST_INCLUDE_DIRS})

set(XSIMD_TESTS
    main.cpp
    xsimd_api_test.hpp
    xsimd_api_test.cpp
    xsimd_algorithms.cpp
    xsimd_basic_test.hpp
    xsimd_basic_test.cpp
    xsimd_basic_math_test.hpp
    xsimd_basic_math_test.cpp
    xsimd_complex_basic_test.hpp
    xsimd_complex_tester.hpp
    xsimd_error_gamma_test.hpp
    xsimd_error_gamma_test.cpp
    xsimd_exponential_test.hpp
    xsimd_exponential_test.cpp
    xsimd_cexponential_test.hpp
    xsimd_cexponential_test.cpp
    xsimd_chyperbolic_test.hpp
    xsimd_chyperbolic_test.cpp
    xsimd_cpower_test.hpp
    xsimd_cpower_test.cpp
    xsimd_ctrigonometric_test.hpp
    xsimd_ctrigonometric_test.cpp
    xsimd_fp_manipulation_test.hpp
    xsimd_fp_manipulation_test.cpp
    xsimd_hyperbolic_test.hpp
    xsimd_hyperbolic_test.cpp
    xsimd_interface_test.cpp
    xsimd_memory_test.cpp
    xsimd_poly_evaluation_test.cpp
    xsimd_poly_evaluation_test.hpp
    xsimd_power_test.hpp
    xsimd_power_test.cpp
    xsimd_rounding_test.hpp
    xsimd_rounding_test.cpp
    xsimd_tester.hpp
    xsimd_test_utils.hpp
    xsimd_trigonometric_test.hpp
    xsimd_trigonometric_test.cpp
)

add_executable(test_xsimd EXCLUDE_FROM_ALL ${XSIMD_TESTS} ${XSIMD_HEADERS})
target_link_libraries(test_xsimd xsimd ${GTEST_BOTH_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT})
target_include_directories(test_xsimd PRIVATE ${XSIMD_INCLUDE_DIR})

if(DEFINED XSIMD_FORCE_X86_INSTR_SET)
    message("Forcing XSIMD_FORCE_X86_INSTR_SET to ${XSIMD_FORCE_X86_INSTR_SET}")
    target_compile_definitions(test_xsimd PRIVATE XSIMD_FORCE_X86_INSTR_SET=${XSIMD_FORCE_X86_INSTR_SET})
endif()

if(DEFINED XSIMD_FORCE_X86_AMD_INSTR_SET)
    message("Forcing XSIMD_FORCE_X86_AMD_INSTR_SET to ${XSIMD_FORCE_X86_AMD_INSTR_SET}")
    target_compile_definitions(test_xsimd PRIVATE XSIMD_FORCE_X86_AMD_INSTR_SET=${XSIMD_FORCE_X86_AMD_INSTR_SET})
endif()

if(DEFINED XSIMD_FORCE_PPC_INSTR_SET)
    message("Forcing XSIMD_FORCE_PPC_INSTR_SET to ${XSIMD_FORCE_PPC_INSTR_SET}")
    target_compile_definitions(test_xsimd PRIVATE XSIMD_FORCE_PPC_INSTR_SET=${XSIMD_FORCE_PPC_INSTR_SET})
endif()

if(DEFINED XSIMD_FORCE_ARM_INSTR_SET)
    message("Forcing XSIMD_FORCE_ARM_INSTR_SET to ${XSIMD_FORCE_ARM_INSTR_SET}")
    target_compile_definitions(test_xsimd PRIVATE XSIMD_FORCE_ARM_INSTR_SET=${XSIMD_FORCE_ARM_INSTR_SET})
endif()

if (CROSS_COMPILE_ARM)
    add_custom_target(xtest COMMAND qemu-arm -L /usr/arm-linux-gnueabi/ test_xsimd DEPENDS test_xsimd)
else()
    add_custom_target(xtest COMMAND test_xsimd DEPENDS test_xsimd)
endif()
