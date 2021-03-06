#------------------------------------------------------------------------------#
# Copyright (c) 2014 Los Alamos National Security, LLC
# All rights reserved.
#------------------------------------------------------------------------------#

cmake_minimum_required(VERSION 3.1)

#~----------------------------------------------------------------------------~#
# VPIC project configuration
#~----------------------------------------------------------------------------~#

project(vpic)

#------------------------------------------------------------------------------#
# If a C++11 compiler is available, then set the appropriate flags
#------------------------------------------------------------------------------#

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

#------------------------------------------------------------------------------#
# Set C flags
#------------------------------------------------------------------------------#

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

#------------------------------------------------------------------------------#
# MPI detection
#------------------------------------------------------------------------------#

find_package(MPI REQUIRED)
list(APPEND MPI_CXX_COMPILE_FLAGS -DOMPI_SKIP_MPICXX -DMPICH_SKIP_MPICXX )
include_directories(${MPI_C_INCLUDE_PATH})

#------------------------------------------------------------------------------#
# Add build options
#------------------------------------------------------------------------------#

option(ENABLE_INTEGRATED_TESTS "enable integrated tests" OFF)

option(USE_V4_ALTIVEC "Enable V4 Altivec" OFF)

option(USE_V4_PORTABLE "Enable V4 Portable" OFF)

option(USE_V4_SSE "Enable V4 SSE" OFF)

option(ENABLE_OPENSSL "Enable OpenSSL support for checksums" OFF)

#------------------------------------------------------------------------------#
# Create include and link aggregates
#
# NOTE: These must be set before creating the compile scripts below.
#------------------------------------------------------------------------------#

string(REPLACE ";" " -I" string_includes "${MPI_C_INCLUDE_PATH}")
if(NOT ${string_includes} STREQUAL "")
  set(VPIC_CXX_FLAGS "-I${string_includes} ${MPI_C_LINK_FLAGS}")
endif(NOT ${string_includes} STREQUAL "")
set(VPIC_CXX_FLAGS "${CMAKE_CXX11_STANDARD_COMPILE_OPTION} ${VPIC_CXX_FLAGS}")

# Add Debug flags to VPIC_CXX_FLAGS
if("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
  set(VPIC_CXX_FLAGS "${VPIC_CXX_FLAGS} ${CMAKE_CXX_FLAGS_DEBUG}")
endif("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")

# Add RelWithDebInfo flags to VPIC_CXX_FLAGS
if("${CMAKE_BUILD_TYPE}" STREQUAL "RelWithDebInfo")
  set(VPIC_CXX_FLAGS "${VPIC_CXX_FLAGS} ${CMAKE_CXX_FLAGS_RELWITHDEBINFO}")
endif("${CMAKE_BUILD_TYPE}" STREQUAL "RelWithDebInfo")

string(REPLACE ";" " " string_libraries "${MPI_CXX_LIBRARIES} ${MPI_C_LIBRARIES}")
set(VPIC_CXX_LIBRARIES "${string_libraries}")

#------------------------------------------------------------------------------#
# OpenSSL
#------------------------------------------------------------------------------#

if(ENABLE_OPENSSL)
  find_package(OpenSSL REQUIRED)

  include_directories(${OPENSSL_INCLUDE_DIR})
  string(REPLACE ";" " " string_libraries "${OPENSSL_LIBRARIES}")
  set(VPIC_CXX_LIBRARIES "${VPIC_CXX_LIBRARIES} ${string_libraries}")
endif(ENABLE_OPENSSL)

find_package(Threads REQUIRED)

#------------------------------------------------------------------------------#
# Act on build options set in project.cmake
#------------------------------------------------------------------------------#

set(USE_V4)
if(USE_V4_ALTIVEC)
  add_definitions(-DUSE_V4_ALTIVEC)
  set(USE_V4 True)
endif(USE_V4_ALTIVEC)

if(USE_V4_PORTABLE)
  add_definitions(-DUSE_V4_PORTABLE)
  set(USE_V4 True)
endif(USE_V4_PORTABLE)

if(USE_V4_SSE)
  add_definitions(-DUSE_V4_SSE)
  set(USE_V4 True)
endif(USE_V4_SSE)

if(ENABLE_OPENSSL)
  add_definitions(-DENABLE_OPENSSL)
endif(ENABLE_OPENSSL)

#------------------------------------------------------------------------------#
# Handle vpic compile script last
#------------------------------------------------------------------------------#

# Allow files to be references from the root, (eg #include "src/vpic/vpic.h")
include_directories(${PROJECT_SOURCE_DIR})

if(BUILD_SHARED_LIBS)
    set(VPIC_CXX_FLAGS "-rdynamic ${VPIC_CXX_FLAGS}")
endif(BUILD_SHARED_LIBS)

if(ENABLE_COVERAGE_BUILD)
    set(VPIC_CXX_FLAGS "${VPIC_CXX_FLAGS} --coverage")
endif(ENABLE_COVERAGE_BUILD)

# process Makefile.run.in to get a simple Makefile.run for a run. Points to
# local built exe wrapper, and has example deck/platform.
configure_file(${CMAKE_SOURCE_DIR}/sample/Makefile.run.in
  ${CMAKE_BINARY_DIR}/bin/Makefile.run)

# install script
configure_file(${CMAKE_SOURCE_DIR}/bin/vpic.in
  ${CMAKE_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/vpic-install)
install(FILES ${CMAKE_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/vpic-install
  DESTINATION bin
  RENAME vpic
  PERMISSIONS
    OWNER_READ OWNER_WRITE OWNER_EXECUTE
    GROUP_READ GROUP_EXECUTE
    WORLD_READ WORLD_EXECUTE
    )

install(FILES ${CMAKE_SOURCE_DIR}/deck/main.cc
  DESTINATION share/vpic)
install(FILES ${CMAKE_SOURCE_DIR}/deck/wrapper.cc
  DESTINATION share/vpic)

# local script
configure_file(${CMAKE_SOURCE_DIR}/bin/vpic-local.in
  ${CMAKE_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/vpic)

file(COPY ${CMAKE_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/vpic
  DESTINATION ${CMAKE_BINARY_DIR}/bin
  FILE_PERMISSIONS
    OWNER_READ OWNER_WRITE OWNER_EXECUTE
    GROUP_READ GROUP_EXECUTE
    WORLD_READ WORLD_EXECUTE
)

#------------------------------------------------------------------------------#
# Add library target
#------------------------------------------------------------------------------#

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${VPIC_CXX_FLAGS}")

file(GLOB_RECURSE VPIC_SRC src/*.c src/*.cc)
file(GLOB_RECURSE VPIC_NOT_SRC src/util/v4/test/v4.cc src/util/rng/test/rng.cc)
list(REMOVE_ITEM VPIC_SRC ${VPIC_NOT_SRC})
option(NO_LIBVPIC "Don't build a libvpic, but all in one" OFF)
if(NO_LIBVPIC)
  set(VPIC_EXPOSE "INTERFACE")
  add_library(vpic INTERFACE)
else()
  set(VPIC_EXPOSE "PRIVATE")
  add_library(vpic ${VPIC_SRC})
  set(VPIC_SRC)
  install(TARGETS vpic LIBRARY DESTINATION lib ARCHIVE DESTINATION lib)
endif()
target_include_directories(vpic INTERFACE ${CMAKE_SOURCE_DIR}/src)
target_link_libraries(vpic ${VPIC_EXPOSE} ${MPI_CXX_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} ${OPENSSL_LIBRARIES} ${CMAKE_DL_LIBS})
target_compile_options(vpic ${VPIC_EXPOSE} ${MPI_C_COMPILE_FLAGS})

macro(build_a_vpic name deck)
  if(NOT EXISTS ${deck})
    message(FATAL_ERROR "Could not find deck '${deck}'")
  endif()
  add_executable(${name} ${CMAKE_SOURCE_DIR}/deck/main.cc ${CMAKE_SOURCE_DIR}/deck/wrapper.cc ${VPIC_SRC})
  target_link_libraries(${name} vpic)
  target_compile_definitions(${name} PRIVATE INPUT_DECK=${deck})
endmacro(build_a_vpic)

if(USER_DECKS)
  foreach(DECK ${USER_DECK})
    get_filename_component(NAME "${DECK}" NAME_WE)
    get_filename_component(DECKDIR "${DECK}" DIRECTORY)
    if(NOT DECKDIR)
      set(DECK "${CMAKE_CURRENT_BINARY_DIR}/${DECK}")
    endif()
    build_a_vpic(${NAME} ${DECK})
  endforeach()
endif()

#------------------------------------------------------------------------------#
# Add VPIC integrated test mechanism
#------------------------------------------------------------------------------#

if(ENABLE_INTEGRATED_TESTS)
  enable_testing()
  add_subdirectory(test/integrated)

endif(ENABLE_INTEGRATED_TESTS)

if(ENABLE_UNIT_TESTS)

  # NOTE: This can also be done by a subdirectory (useful when we have multiple tests)
  # Add includes for Catch2
  set(TEST_DIR "./test/include")
  include_directories(${TEST_DIR})

  # Add tests individually
  if (USE_V4)
    add_executable(v4 src/util/v4/test/v4.cc)
    add_test(NAME v4 COMMAND ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 1 ${MPIEXEC_PREFLAGS} ./v4)
  endif(USE_V4)

  # RNG tests
  add_executable(rng src/util/rng/test/rng.cc)
  target_link_libraries(rng vpic)
  add_test(NAME rng COMMAND ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 1 ${MPIEXEC_PREFLAGS} ./rng)

endif(ENABLE_UNIT_TESTS)
#~---------------------------------------------------------------------------~-#
# vim: set tabstop=2 shiftwidth=2 expandtab :
#~---------------------------------------------------------------------------~-#
