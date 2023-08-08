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
find_package(MPI 3.0 REQUIRED)


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

add_library(walberla_bridge SHARED)
add_library(walberla_bridge::walberla ALIAS walberla_bridge)

set_target_properties(walberla_bridge PROPERTIES CXX_CLANG_TIDY "")
target_include_directories(walberla_bridge PUBLIC include)

add_library(walberla_bridge_cpp_flags INTERFACE)
set_target_properties(walberla_bridge_cpp_flags PROPERTIES CXX_CLANG_TIDY "")
add_library(walberla_bridge::walberla::cpp_flags ALIAS walberla_bridge_cpp_flags)
if(WALBERLA_BRIDGE_BUILD_WITH_WALBERLA_AVX)
  target_link_libraries(walberla_bridge_cpp_flags
                        INTERFACE walberla_bridge::avx_flags)
endif()
install(TARGETS walberla_bridge
        LIBRARY DESTINATION ${WALBERLA_BRIDGE_INSTALL_PYTHON}/walberla_bridge)

if(WALBERLA_BRIDGE_BUILD_WITH_CUDA AND WALBERLA_BUILD_WITH_CUDA)
  walberla_bridge_add_gpu_library(walberla_bridge_cuda SHARED)
  add_library(walberla_bridge::walberla_cuda ALIAS walberla_bridge_cuda)
  target_link_libraries(walberla_bridge_cuda PRIVATE CUDA::cuda_driver
                                                       CUDA::cudart)
  target_link_libraries(walberla_bridge_cuda PUBLIC walberla_bridge::utils
                        PRIVATE ${WALBERLA_LIBS})
  target_include_directories(walberla_bridge_cuda PUBLIC include)
  target_include_directories(
    walberla_bridge_cuda PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}
    PRIVATE ${WALBERLA_INCLUDE_DIRS} ${walberla_BINARY_DIR}/src)
  install(TARGETS walberla_bridge_cuda
          LIBRARY DESTINATION ${WALBERLA_BRIDGE_INSTALL_PYTHON}/walberla_bridge)
  target_link_libraries(walberla_bridge PUBLIC walberla_bridge::walberla_cuda)
endif()

target_link_libraries(
  walberla_bridge PUBLIC MPI::MPI_CXX walberla_bridge::utils
  PRIVATE Boost::boost walberla_bridge::cpp_flags walberla_bridge::walberla::cpp_flags
          ${WALBERLA_LIBS})
target_include_directories(
  walberla_bridge PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}
  PRIVATE ${WALBERLA_INCLUDE_DIRS} ${walberla_BINARY_DIR}/src)

add_subdirectory(src)

if(WALBERLA_BRIDGE_BUILD_TESTS)
  add_subdirectory(tests)
endif()
