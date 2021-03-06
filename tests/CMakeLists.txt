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


message(STATUS "Building tests")

find_package (Boost COMPONENTS unit_test_framework REQUIRED)

include_directories ( ${Boost_INCLUDE_DIRS} )
include_directories(
  ${CMAKE_SOURCE_DIR}/include
  ${CMAKE_SOURCE_DIR}/include/fallback
  ${PROJECT_BINARY_DIR})

add_definitions (-DBOOST_TEST_DYN_LINK)

set(test_src
  test_epu.cpp test_perm16.cpp test_perm_all.cpp test_bmat8.cpp)
foreach(f ${test_src})
  get_filename_component(testName ${f} NAME_WE)
  add_executable (${testName} ${f})
  target_link_libraries(${testName} ${Boost_UNIT_TEST_FRAMEWORK_LIBRARY} )
endforeach(f)

# Test for multiple inclusions
add_executable (test_mincl test_mincl.cpp test_mincl0.cpp test_mincl1.cpp)
target_link_libraries(test_mincl ${Boost_UNIT_TEST_FRAMEWORK_LIBRARY} )

add_test (TestEPU test_epu)
add_test (TestPerm16 test_perm16)
add_test (TestPermAll test_perm_all)
add_test (TestBMat8 test_bmat8)
add_test (TestMultIncl test_mincl)

