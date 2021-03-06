#****************************************************************************#
#       Copyright (C) 2016 Florent Hivert <Florent.Hivert@lri.fr>,           #
#                                                                            #
#  Distributed under the terms of the GNU General Public License (GPL)       #
#                                                                            #
#    This code is distributed in the hope that it will be useful,            #
#    but WITHOUT ANY WARRANTY; without even the implied warranty of          #
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU       #
#   General Public License for more details.                                 #
#                                                                            #
#  The full text of the GPL is available at:                                 #
#                                                                            #
#                  http://www.gnu.org/licenses/                               #
#****************************************************************************#

cmake_minimum_required(VERSION 2.8)

#####################
# Project description
project(HPCombi)

set(DESCRIPTION  "High Performance Combinatorics in C++ using vector instructions"
  CACHE STRING "Project description.")
set(VERSION_MAJOR   0   CACHE STRING "Project major version number.")
set(VERSION_MINOR   0   CACHE STRING "Project minor version number.")
set(VERSION_PATCH   6   CACHE STRING "Project patch version number.")
mark_as_advanced(VERSION_MAJOR VERSION_MINOR VERSION_PATCH)

message(STATUS "**** Build type = ${CMAKE_BUILD_TYPE}")
# set(CMAKE_BUILD_TYPE Release)
# set(CMAKE_BUILD_TYPE Debug)
# set(CMAKE_VERBOSE_MAKEFILE 1)


################################
# General compiler configuration
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_EXTENSIONS OFF) # -std=c++11 instead of -std=gnu++11

add_definitions(-DHPCOMBI_HAVE_CONFIG)

message(STATUS "*** Compiler id is ${CMAKE_CXX_COMPILER_ID}")
if ( CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang" )
  # Workaround of CMAKE bug https://stackoverflow.com/questions/47213356/
  set(CMAKE_REQUIRED_FLAGS ${CMAKE_REQUIRED_FLAGS} -std=c++11)
  add_compile_options(-std=c++11 -Wall -g -pg)
endif()


##################################
# Checks for compiler capabilities
include(CheckCXXSymbolExists)
include(CheckIncludeFileCXX)
include(CheckCXXSourceCompiles)
include(CheckCXXCompilerFlag)

## Check for static lcm
check_include_file_cxx("experimental/numeric" HPCOMBI_HAVE_EXPERIMENTAL_NUMERIC)
if (HPCOMBI_HAVE_EXPERIMENTAL_NUMERIC)
  check_cxx_source_compiles(
        "
        #include <experimental/numeric>
        static_assert(std::experimental::lcm(4, 6) == 12, \"Buggy lcm\");
        int main() { }
        "
        HPCOMBI_HAVE_EXPERIMENTAL_NUMERIC_LCM)
endif (HPCOMBI_HAVE_EXPERIMENTAL_NUMERIC)

## Check for buggy constexpr support G++ 5.0
check_cxx_source_compiles(
  "
  using T = int;
  constexpr int exec(T f()) { return f(); }
  constexpr int foo() { return 1; }
  static_assert(exec(foo) == 1, \"Failed exec\");
  int main() {}
  "
  HPCOMBI_CONSTEXPR_FUN_ARGS)

check_cxx_compiler_flag('-mavx' HPCOMBI_HAVE_FLAG_AVX)
#check_cxx_compiler_flag('-march=native' HPCOMBI_HAVE_FLAG_NATIVE)
#check_cxx_compiler_flag('-mavx2' HPCOMBI_HAVE_FLAG_AVX2)
#check_cxx_compiler_flag('-mavx512bw' HPCOMBI_HAVE_FLAG_AVX512BW)

## Check for Intel intrisics
check_include_file_cxx("x86intrin.h" HPCOMBI_HAVE_X86INTRIN)
if (NOT ${HPCOMBI_HAVE_FLAG_AVX} OR NOT ${HPCOMBI_HAVE_X86INTRIN})
  message(FATAL_ERROR "No SSE/AVX compiler intrinsics")
endif()
file(READ ${CMAKE_SOURCE_DIR}/list_intrin.txt hpcombi_intrinsics)
string(REPLACE ";" "|" hpcombi_intrinsics "${hpcombi_intrinsics}")
string(REPLACE "\n" ";" hpcombi_intrinsics "${hpcombi_intrinsics}")
foreach (intrin ${hpcombi_intrinsics})
  if ("${intrin}" MATCHES "^#" ) # Comment
    continue()
  endif()
  string(REPLACE "|" ";" intrin "${intrin}")
  list(GET intrin 0 intrin_name)
  list(GET intrin 1 intrin_params)
  set(CMAKE_REQUIRED_FLAGS "-mavx")
  check_cxx_source_compiles("
    #include<x86intrin.h>
    int main() {
      ${intrin_name}(${intrin_params});
      return 0;
    }
  "
  "HPCOMBI_HAVE_${intrin_name}"
  )
  if (NOT "${HPCOMBI_HAVE_${intrin_name}}")
    message(FATAL_ERROR "Intrinsic ${intrin_name} not supported by compiler")
  endif()
endforeach()

add_compile_options(-mavx -mtune=native -funroll-loops -flax-vector-conversions)

###################
# Project Structure
add_subdirectory(examples)
add_subdirectory(benchmark)
add_subdirectory(doc)

include_directories(
  ${CMAKE_SOURCE_DIR}/include
  ${PROJECT_BINARY_DIR})


#########
# Testing

IF (BUILD_TESTING)
  include(CTest)
  enable_testing ()
  add_subdirectory(tests)
ENDIF(BUILD_TESTING)

#####################
# config.h file stuff
configure_file(${CMAKE_SOURCE_DIR}/config.h.in ${CMAKE_BINARY_DIR}/HPCombi-config.h)
configure_file(${CMAKE_SOURCE_DIR}/VERSION.in ${CMAKE_BINARY_DIR}/VERSION)

set(AUTOGENERATED_WARNING "WARNING: THIS IS A CMAKE AUTO-GENERATED FILE.")

####################
# Installation

set(install_misc README.md LICENSE list_intrin.txt)
foreach(f ${install_misc})
  install (FILES ${CMAKE_SOURCE_DIR}/${f}
    DESTINATION share/${CMAKE_PROJECT_NAME})
endforeach(f)

install (FILES ${CMAKE_CURRENT_BINARY_DIR}/VERSION DESTINATION ".")

install (FILES ${CMAKE_CURRENT_BINARY_DIR}/HPCombi-config.h
  DESTINATION include/${CMAKE_PROJECT_NAME})

install (
  DIRECTORY ${CMAKE_SOURCE_DIR}/include/
  DESTINATION include/${CMAKE_PROJECT_NAME}
  FILES_MATCHING PATTERN "*.hpp")

###################
# pkgconfig stuff
# configure_file(${CMAKE_CURRENT_SOURCE_DIR}/hpcombi.pc.in
#                ${CMAKE_CURRENT_BINARY_DIR}/hpcombi.pc @ONLY)


#################
# Packing stuff
#
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${DESCRIPTION}")
set(CPACK_PACKAGE_VENDOR "Florent Hivert <florent.hivert@lri.fr>")
set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/README.md")
set(CPACK_PACKAGE_VERSION_MAJOR "${VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${VERSION_MINOR}")
set(CPACK_PACKAGE_VERSION_PATCH "${VERSION_PATCH}")

set(CPACK_GENERATOR "TGZ")
SET(CPACK_PACKAGE_FILE_NAME
  "HPCombi-${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}")
include(CPack)

########################
# Custom target for TAGS
if (UNIX)
  add_custom_target(tags etags --members --declarations  `find ${CMAKE_SOURCE_DIR}/ -name *.cpp -or -name *.hpp -or -name *.c -or -name *.h` -o ${CMAKE_SOURCE_DIR}/TAGS)
  add_custom_target(etags DEPENDS tags)
endif (UNIX)



