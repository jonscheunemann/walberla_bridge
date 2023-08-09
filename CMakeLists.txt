#
# Copyright (C) 2020-2023 The ESPResSo project
#
# This file is part of ESPResSo.
#
# ESPResSo is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
#
# ESPResSo is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.
#

cmake_minimum_required(VERSION 3.20)
project(walberla_bridge)
message(STATUS "CMake version: ${CMAKE_VERSION}")
if(POLICY CMP0076)
  # make target_sources() convert relative paths to absolute
  cmake_policy(SET CMP0076 NEW)
endif()
if(POLICY CMP0025)
  # make CXX_COMPILER_ID return "AppleClang" instead of "Clang" for Apple Clang
  cmake_policy(SET CMP0025 NEW)
endif()
if(POLICY CMP0074)
  # make find_package() use <PackageName>_ROOT variables
  cmake_policy(SET CMP0074 NEW)
endif()

# C++ standard
enable_language(CXX)
set(CMAKE_CXX_STANDARD 17 CACHE STRING "C++ standard to be used")
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

find_package(MPI 3.0 REQUIRED)
list(APPEND BOOST_COMPONENTS mpi serialization filesystem system)

#Options for caching
option(WALBERLA_BRIDGE_BUILD_WITH_CCACHE "Use ccache compiler invocation." OFF)
if(WALBERLA_BRIDGE_BUILD_WITH_CCACHE)
  find_program(CCACHE_PROGRAM ccache REQUIRED)
  if(CCACHE_PROGRAM)
    message(STATUS "Found ccache: ${CCACHE_PROGRAM}")
    set(CMAKE_CXX_COMPILER_LAUNCHER ${CCACHE_PROGRAM})
    set(CMAKE_CUDA_COMPILER_LAUNCHER ${CCACHE_PROGRAM})
  endif()
endif()


if(WALBERLA_BRIDGE_BUILD_TESTS)
  enable_testing()
  add_custom_target(check)
  set(WALBERLA_BRIDGE_CTEST_ARGS ""
      CACHE STRING
            "Extra arguments to give to ctest calls (separated by semicolons)")
  set(WALBERLA_BRIDGE_TEST_NP "4" CACHE STRING
                                "Maximal number of MPI ranks to use per test")
  if(WALBERLA_BRIDGE_BUILD_WITH_PYTHON)
    add_subdirectory(testsuite) #AAAAAAAAAAAAAAAAAAAAAH
  endif()

# Target for the unit tests
  add_custom_target(
    check_unit_tests
    COMMAND ${CMAKE_CTEST_COMMAND} --timeout ${WALBERLA_BRIDGE_TEST_TIMEOUT}
            ${WALBERLA_BRIDGE_CTEST_ARGS} --output-on-failure)
  list(APPEND BOOST_COMPONENTS unit_test_framework)
  add_dependencies(check check_unit_tests)
endif()

find_package(Boost 1.74.0 REQUIRED ${BOOST_COMPONENTS})

add_library(walberla_bridge_cpp_flags INTERFACE)
add_library(walberla_bridge::cpp_flags ALIAS walberla_bridge_cpp_flags)
target_compile_options(
  walberla_bridge_cpp_flags
  INTERFACE
    -Wall
    -Wextra
    -pedantic
    $<$<BOOL:${WALBERLA_BRIDGE_WARNINGS_ARE_ERRORS}>:-Werror>
    # add extra warnings
    $<$<CXX_COMPILER_ID:Clang>:-Wextern-initializer>
    $<$<CXX_COMPILER_ID:Clang>:-Wrange-loop-analysis>
    -Wfloat-conversion
    $<$<CXX_COMPILER_ID:Clang,AppleClang,IntelLLVM>:-Wimplicit-float-conversion>
    $<$<CXX_COMPILER_ID:Clang,AppleClang,IntelLLVM>:-Wunused-exception-parameter>
    $<$<CXX_COMPILER_ID:Clang,AppleClang,IntelLLVM>:-Wmissing-variable-declarations>
    $<$<AND:$<CXX_COMPILER_ID:Clang>,$<VERSION_GREATER_EQUAL:$<CXX_COMPILER_VERSION>,11.0.0>>:-Wnon-c-typedef-for-linkage>
    $<$<NOT:$<CXX_COMPILER_ID:Intel>>:-Wdelete-non-virtual-dtor>
    # disable warnings from -Wextra
    -Wno-sign-compare
    -Wno-unused-function
    -Wno-unused-parameter
    $<$<CXX_COMPILER_ID:GNU>:-Wno-clobbered>
    $<$<CXX_COMPILER_ID:Intel,IntelLLVM>:-diag-disable=592>
    $<$<CXX_COMPILER_ID:Clang,AppleClang>:-Wno-gnu-zero-variadic-macro-arguments>
    $<$<AND:$<CXX_COMPILER_ID:GNU>,$<VERSION_GREATER_EQUAL:$<CXX_COMPILER_VERSION>,8.1.0>>:-Wno-cast-function-type>
    $<$<NOT:$<CXX_COMPILER_ID:Intel,IntelLLVM>>:-Wno-implicit-fallthrough>
    $<$<NOT:$<CXX_COMPILER_ID:Intel,IntelLLVM,GNU>>:-Wno-unused-private-field>)


# cmake-format: off
include(FetchContent)
FetchContent_Declare(
  walberla
  GIT_REPOSITORY https://i10git.cs.fau.de/walberla/walberla.git
  GIT_TAG        065ce5f311850371a97ac4766f47dbb5ca8424ba
)
# workaround for https://gitlab.kitware.com/cmake/cmake/-/issues/21146
if(NOT DEFINED walberla_SOURCE_DIR OR NOT EXISTS "${walberla_SOURCE_DIR}")
  FetchContent_Populate(walberla)
endif()
set(WALBERLA_BUILD_TESTS off CACHE BOOL "")
set(WALBERLA_BUILD_BENCHMARKS off CACHE BOOL "")
set(WALBERLA_BUILD_TOOLS off CACHE BOOL "")
set(WALBERLA_BUILD_TUTORIALS off CACHE BOOL "")
set(WALBERLA_BUILD_SHOWCASES off CACHE BOOL "")
set(WALBERLA_BUILD_DOC off CACHE BOOL "")
set(WALBERLA_LOGLEVEL "WARNING" CACHE STRING "")
set(CMAKE_POSITION_INDEPENDENT_CODE on CACHE BOOL "")
if(WALBERLA_BRIDGE_BUILD_WITH_CUDA)
  set(WALBERLA_BUILD_WITH_CUDA "on" CACHE BOOL "")
  if(CMAKE_VERSION VERSION_LESS 3.25 OR NOT WALBERLA_BRIDGE_CUDA_COMPILER STREQUAL
                                        "clang")
    if(NOT DEFINED CMAKE_CUDA_ARCHITECTURES)
      set(CMAKE_CUDA_ARCHITECTURES 75)
    endif()
  endif()
endif()
if(WALBERLA_BRIDGE_BUILD_WITH_WALBERLA_FFT)
  set(WALBERLA_BRIDGE_USE_WALBERLA_FFT 1)
  set(WALBERLA_BUILD_WITH_FFTW on CACHE BOOL "")
else()
  set(WALBERLA_BUILD_WITH_FFTW off CACHE BOOL "")
endif()
set(WALBERLA_BUILD_WITH_FASTMATH off CACHE BOOL "")
add_subdirectory("${walberla_SOURCE_DIR}" "${walberla_BINARY_DIR}")
set(WALBERLA_LIBS
    walberla::core walberla::domain_decomposition walberla::blockforest
    walberla::boundary walberla::field walberla::lbm walberla::timeloop
    walberla::vtk)
if(WALBERLA_BRIDGE_BUILD_WITH_WALBERLA_FFT)
  set(WALBERLA_LIBS ${WALBERLA_LIBS} walberla::fft)
endif()
if(WALBERLA_BRIDGE_BUILD_WITH_CUDA AND WALBERLA_BUILD_WITH_CUDA)
  set(WALBERLA_LIBS ${WALBERLA_LIBS} walberla::cuda)
endif()
# workaround for https://gitlab.kitware.com/cmake/cmake/-/issues/21283
foreach(target_w_namespace IN LISTS WALBERLA_LIBS)
string(REPLACE "walberla::" "" target_wo_namespace ${target_w_namespace})
add_library(${target_w_namespace} ALIAS ${target_wo_namespace})
endforeach()
if(WALBERLA_BRIDGE_BUILD_WITH_WALBERLA_AVX)
function(walberla_bridge_avx_flags_callback COMPILER_AVX2_FLAG)
  target_compile_options(
    walberla_bridge_avx_flags INTERFACE "${COMPILER_AVX2_FLAG}"
                                 "-DESPRESSO_BUILD_WITH_AVX_KERNELS")
endfunction()
walberla_bridge_enable_avx2_support(walberla_bridge_avx_flags_callback)
endif()

add_subdirectory(src)
add_subdirectory(include)

if(WALBERLA_BRIDGE_BUILD_TESTS)
  add_subdirectory(tests)
endif()
