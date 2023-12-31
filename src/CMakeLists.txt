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


add_library(walberla_bridge SHARED)
add_library(walberla_bridge::walberla ALIAS walberla_bridge)

set_target_properties(walberla_bridge PROPERTIES CXX_CLANG_TIDY "")
target_include_directories(walberla_bridge PUBLIC ../include)

add_library(walberla_bridge_walberla_cpp_flags INTERFACE)
set_target_properties(walberla_bridge_walberla_cpp_flags PROPERTIES CXX_CLANG_TIDY "")
add_library(walberla_bridge::walberla::cpp_flags ALIAS walberla_bridge_walberla_cpp_flags)
target_link_libraries(walberla_bridge PUBLIC walberla_bridge::utils)
if(WALBERLA_BRIDGE_BUILD_WITH_WALBERLA_AVX)
  target_link_libraries(walberla_bridge_walberla_cpp_flags
                        INTERFACE walberla_bridge::avx_flags)
endif()

if(WALBERLA_BRIDGE_BUILD_WITH_CUDA AND WALBERLA_BUILD_WITH_CUDA)
  walberla_bridge_add_gpu_library(walberla_bridge_cuda SHARED)
  add_library(walberla_bridge::walberla_cuda ALIAS walberla_bridge_cuda)
  target_link_libraries(walberla_bridge_cuda PRIVATE CUDA::cuda_driver
                                                       CUDA::cudart)
  target_link_libraries(walberla_bridge_cuda PUBLIC walberla_bridge::utils
                        PRIVATE ${WALBERLA_LIBS})
  target_include_directories(walberla_bridge_cuda PUBLIC ../include)
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


add_subdirectory(electrokinetics)
add_subdirectory(lattice_boltzmann)

target_sources(walberla_bridge PRIVATE LatticeModel.cpp LatticeWalberla.cpp
                                         walberla_init.cpp)

install(TARGETS walberla_bridge
        LIBRARY DESTINATION ${WALBERLA_BRIDGE_INSTALL_PYTHON}/walberla_bridge)