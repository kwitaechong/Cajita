set(GTEST_SOURCE_DIR ${CMAKE_SOURCE_DIR}/tpls)

include_directories(${GTEST_SOURCE_DIR})
add_library(Cajita_gtest ${GTEST_SOURCE_DIR}/gtest/gtest-all.cc)
set_target_properties(Cajita_gtest PROPERTIES
    CXX_STANDARD 11
    CXX_STANDARD_REQUIRED YES
    CXX_EXTENSIONS NO
    )

include_directories(${CMAKE_CURRENT_BINARY_DIR})
include_directories(${CMAKE_CURRENT_SOURCE_DIR})

set(gtest_args --gtest_color=yes)

##--------------------------------------------------------------------------##
## General tests.
##--------------------------------------------------------------------------##
foreach(_test CajitaVersion)
  add_executable(${_test}_test tst${_test}.cpp unit_test_main.cpp)
  target_link_libraries(${_test}_test Cajita Cajita_gtest)
  add_test(NAME ${_test}_test COMMAND ${_test}_test --gtest_color=yes)
endforeach()

##--------------------------------------------------------------------------##
## On-node tests with and without MPI.
##--------------------------------------------------------------------------##
set(CAJITA_SUPPORTED_DEVICES Serial OpenMP Cuda)

macro(Cajita_add_tests)
  cmake_parse_arguments(CAJITA_UNIT_TEST "MPI" "" "NAMES" ${ARGN})
  set(CAJITA_UNIT_TEST_MPIEXEC_NUMPROCS 1)
  foreach( _np 2 4 8 )
    if(MPIEXEC_MAX_NUMPROCS GREATER_EQUAL ${_np})
      list(APPEND CAJITA_UNIT_TEST_MPIEXEC_NUMPROCS ${_np})
    endif()
  endforeach()
  set(CAJITA_UNIT_TEST_NUMTHREADS 1)
  foreach( _nt 2 4 8 )
    if(MPIEXEC_MAX_NUMPROCS GREATER_EQUAL ${_nt})
      list(APPEND CAJITA_UNIT_TEST_NUMTHREADS ${_nt})
    endif()
  endforeach()
  set(CAJITA_UNIT_TEST_MAIN unit_test_main.cpp)
  foreach(_device ${CAJITA_SUPPORTED_DEVICES})
    if(Kokkos_DEVICES MATCHES "${_device}")
      set(_dir ${CMAKE_CURRENT_BINARY_DIR}/${_device})
      file(MAKE_DIRECTORY ${_dir})
      foreach(_test ${CAJITA_UNIT_TEST_NAMES})
        set(_file ${_dir}/tst${_test}_${_device}.cpp)
        file(WRITE ${_file} "#include <Test${_device}_Category.hpp>\n")
        file(APPEND ${_file} "#include <tst${_test}.hpp>\n")
        set(_target ${_test}_test_${_device})
        add_executable(${_target} ${_file} ${CAJITA_UNIT_TEST_MAIN})
        target_include_directories(${_target} PUBLIC ${_dir})
        target_link_libraries(${_target} Cajita Cajita_gtest)
        if(CAJITA_UNIT_TEST_MPI)
          foreach(_np ${CAJITA_UNIT_TEST_MPIEXEC_NUMPROCS})
            add_test(NAME ${_target}_${_np} COMMAND
              ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} ${_np} ${MPIEXEC_PREFLAGS}
              ${_target} ${MPIEXEC_POSTFLAGS} ${gtest_args})
          endforeach()
        else()
          if(_device STREQUAL OpenMP)
            foreach(_thread ${CAJITA_UNIT_TEST_NUMTHREADS})
              add_test(NAME ${_target}_${_thread} COMMAND
                ${_target} ${gtest_args} --kokkos-threads=${_thread})
            endforeach()
          else()
            add_test(NAME ${_target} COMMAND ${_target} ${gtest_args})
          endif()
        endif()
      endforeach()
    endif()
  endforeach()
endmacro()

Cajita_add_tests(NAMES IndexSpace)
Cajita_add_tests(MPI NAMES GlobalGrid Block Array Halo StructuredSolver BovWriter)