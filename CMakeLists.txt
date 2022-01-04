################################################################################
# CMakeLists.txt
#
# Root CMake build script for generator
#
# Copyright (C) 2016-2017 Sebastian Lamm <lamm@ira.uka.de>
#
# All rights reserved. Published under the BSD-2 license in the LICENSE file.
################################################################################

cmake_minimum_required(VERSION 3.14)

project(kagen LANGUAGES CXX)
#set_target_properties(kagen PROPERTIES LINKER_LANGUAGE CXX)

# prohibit in-source builds
if("${PROJECT_SOURCE_DIR}" STREQUAL "${PROJECT_BINARY_DIR}")
  message(SEND_ERROR "In-source builds are not allowed.")
endif()

# default to Debug building for single-config generators
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message("Defaulting CMAKE_BUILD_TYPE to Release")
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Build type")
endif()

option(KAGEN_USE_LTO "Compile with -flto (link-time optimization)." OFF)

option(KAGEN_OUTPUT_EDGES "Output edge list." ON)
option(KAGEN_SINGLE_OUTPUT "Output single edge list." ON)
option(KAGEN_OMIT_HEADER "Omit header in edge list" OFF)
option(KAGEN_BINARY_OUTPUT "Output edge lists in binary format" ON)

################################################################################

# variables to collect compile-time definitions, include dirs, and libraries

if(KAGEN_OUTPUT_EDGES)
    list(APPEND KAGEN_DEFINITIONS OUTPUT_EDGES)
  if (KAGEN_SINGLE_OUTPUT)
      list(APPEND KAGEN_DEFINITIONS SINGLE_LIST)
  endif()
  if (KAGEN_OMIT_HEADER)
      list(APPEND KAGEN_DEFINITIONS OMIT_HEADER)
  endif()
  if (KAGEN_BINARY_OUTPUT)
      list(APPEND KAGEN_DEFINITIONS BINARY_OUT)
  endif()
endif()

# find MPI
find_package(MPI REQUIRED)

# find CGAL
set(CGAL_DO_NOT_WARN_ABOUT_CMAKE_BUILD_TYPE TRUE)
find_package(CGAL REQUIRED)

# use sampling lib and libmorton
add_subdirectory(${CMAKE_CURRENT_LIST_DIR}/extlib/sampling EXCLUDE_FROM_ALL)
add_subdirectory(${CMAKE_CURRENT_LIST_DIR}/extlib/libmorton EXCLUDE_FROM_ALL)

set(kagen_SOURCE_DIR ${CMAKE_CURRENT_LIST_DIR})
set(KAGEN_TARGET_INCLUDE_DIRS
  ${kagen_SOURCE_DIR}/extlib/sampling/sampling
  ${kagen_SOURCE_DIR}/app
  ${kagen_SOURCE_DIR}/include
  ${kagen_SOURCE_DIR}/include/generators
  ${kagen_SOURCE_DIR}/include/generators/barabassi
  ${kagen_SOURCE_DIR}/include/generators/geometric
  ${kagen_SOURCE_DIR}/include/generators/geometric/delaunay
  ${kagen_SOURCE_DIR}/include/generators/geometric/rgg
  ${kagen_SOURCE_DIR}/include/generators/gnm
  ${kagen_SOURCE_DIR}/include/generators/gnp
  ${kagen_SOURCE_DIR}/include/generators/hyperbolic
  ${kagen_SOURCE_DIR}/include/io
  ${kagen_SOURCE_DIR}/include/tools
  ${kagen_SOURCE_DIR}/interface/)
add_library(libkagen INTERFACE)
target_include_directories(libkagen SYSTEM INTERFACE ${KAGEN_TARGET_INCLUDE_DIRS})
target_link_libraries(libkagen INTERFACE 
    sampling 
    tlx 
    CGAL::CGAL 
    libmorton::libmorton spooky MPI::MPI_CXX)
target_compile_definitions(libkagen INTERFACE ${KAGEN_DEFINITIONS})
target_compile_definitions(libkagen INTERFACE BOOST_BIND_GLOBAL_PLACEHOLDERS)
check_cxx_compiler_flag("-march=native" KAGEN_MARCH_NATIVE)
if (KAGEN_MARCH_NATIVE)
    message(STATUS "Using -march=native")
    target_compile_options(libkagen INTERFACE "-march=native")
endif()
if (KAGEN_USE_LTO)
    message(STATUS "Using link time optimization")
    set_target_properties(libkagen PROPERTIES INTERPROCEDURAL_OPTIMIZATION TRUE) 
endif()

################################################################################
### Build Macros

# macro for building main programs with correct libraries
macro(do_build TARGETNAME)
  add_executable(${TARGETNAME} ${ARGN})
  target_link_libraries(${TARGETNAME} libkagen)
  #target_include_directories(${TARGETNAME} PUBLIC ${KAGEN_INCLUDE_DIRS})
  #target_link_libraries(${TARGETNAME} ${KAGEN_LINK_LIBRARIES})
endmacro(do_build)

# macro for building main programs with correct libraries
macro(mpi_build TARGETNAME)
  add_executable(${TARGETNAME} ${ARGN})
  target_link_libraries(${TARGETNAME} libkagen)
  #target_include_directories(${TARGETNAME} PUBLIC ${KAGEN_INCLUDE_DIRS})
  #target_link_libraries(${TARGETNAME} ${KAGEN_LINK_LIBRARIES} ${MPI_LIBRARIES})
endmacro(mpi_build)

# macro for building a single-source thrill program
macro(build_prog PROGNAME)
  string(REPLACE "/" "_" TARGETNAME "${PROGNAME}") # replace slashes
  do_build(${TARGETNAME} ${PROGNAME}.cpp ${ARGN})
endmacro(build_prog)

macro(build_mpi_prog PROGNAME)
  string(REPLACE "/" "_" TARGETNAME "${PROGNAME}") # replace slashes
  mpi_build(${TARGETNAME} ${PROGNAME}.cpp ${ARGN})
endmacro(build_mpi_prog)

################################################################################

# descend into library source
# add_subdirectory(include)

# descend into apps
add_subdirectory(app)

################################################################################
