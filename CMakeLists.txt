########################################################
#
#    Copyright (c) 2014-2015
#      SMASH Team
#
#    BSD 3-clause license
#
#########################################################

# Minimum cmake version this is tested on
cmake_minimum_required(VERSION 3.20)

# The name, version and language of our project
project(SMASH VERSION 2.2.1 LANGUAGES CXX)

# Fail if cmake is called in the source directory
if(CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR)
   message(FATAL_ERROR "In-source build is forbidden!")
endif()

# Tell cmake where to find our modules
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# Check python stuff needed for smashAnalysis and smashVHLLEhybrid
include(FindPythonModules)
find_package(PythonInterp 3.8 REQUIRED)
find_package(PythonLibs 3.8)
find_python_module(numpy VERSION 1.17.4 REQUIRED)
find_python_module(scipy VERSION 1.3.3 REQUIRED)
find_python_module(matplotlib VERSION 3.3.1 REQUIRED)
find_python_module(argparse VERSION 1.1 REQUIRED)
find_python_module(yaml VERSION 5.3.1 REQUIRED)
find_python_module(pandas VERSION 0.25.3 REQUIRED)

# Set a default value for CMAKE_BUILD_TYPE, otherwise we get something
# which is none of the options.
if(NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING
      "Choose the type of build, options are: Debug Release RelWithDebInfo Profiling."
      FORCE)
endif(NOT CMAKE_BUILD_TYPE)

# We use little endian. Create variable since it is needed in pdgcode.h
add_definitions("-DLITTLE_ENDIAN_ARCHITECTURE")

include_directories(SYSTEM 
                    ${BOOST_ROOT}/include
                    ${CUBA_ROOT}/include
                    ${EINHARD_ROOT}/include
                    ${EINHARD_ROOT}/include/einhard
                    ${FASTJET_ROOT}/include
                    ${GSL_ROOT_DIR}/include
                    ${ROOTSYS}/include
                    ${YAML_CPP_ROOT}/include
                    ${YODA_ROOT}/include)

link_directories(${CUBA_ROOT}/lib
                 ${EINHARD_ROOT}/lib
                 ${FASTJET_ROOT}/lib
                 ${ROOTSYS}/lib
                 ${YAML_CPP_ROOT}/lib64
                 ${YODA_ROOT}/lib)

# set up the profiling build type
set(CMAKE_CXX_FLAGS_PROFILING "-O3 -DNDEBUG -pg" CACHE STRING "Flags used by the compiler during profile builds." FORCE)
set(CMAKE_C_FLAGS_PROFILING "-O3 -DNDEBUG -pg" CACHE STRING "Flags used by the compiler during profile builds." FORCE)
set(CMAKE_EXE_LINKER_FLAGS_PROFILING "${CMAKE_EXE_LINKER_FLAGS_RELEASE} -pg" CACHE STRING "Flags used by the compiler during profile builds." FORCE)
mark_as_advanced(CMAKE_CXX_FLAGS_PROFILING CMAKE_C_FLAGS_PROFILING CMAKE_EXE_LINKER_FLAGS_RELEASE)

# enable standard CTest
# tests are enabled/disabled with the BUILD_TESTING switch to cmake
# uncomment the next line to default BUILD_TESTING to OFF
option(BUILD_TESTING "Build the testing tree." OFF)
include(CTest)

# SMASH
add_subdirectory(src/smash)
# hadron sampler
add_subdirectory(src/hadron-sampler)

INSTALL(DIRECTORY share/smashAnalysis/ DESTINATION share/smashAnalysis)
INSTALL(DIRECTORY share/smashVHLLEhybrid/ DESTINATION share/smashVHLLEhybrid)
INSTALL(DIRECTORY input/ DESTINATION share/smash)
INSTALL(PROGRAMS scripts/create_report.sh scripts/set_up_hybrid.sh scripts/slurm-job.sh
                 scripts/submit_to_slurm.sh scripts/update_config_version.sh DESTINATION bin)
# reconfigure scripts and place to correct directory
configure_file(src/smashAnalysis.sh.in ${CMAKE_CURRENT_BINARY_DIR}/smashAnalysis.sh @ONLY USE_SOURCE_PERMISSIONS)
configure_file(src/smashPaths.sh.in ${CMAKE_CURRENT_BINARY_DIR}/smashPaths.sh @ONLY USE_SOURCE_PERMISSIONS)
configure_file(src/smashVHLLEhybrid.sh.in ${CMAKE_CURRENT_BINARY_DIR}/smashVHLLEhybrid.sh @ONLY USE_SOURCE_PERMISSIONS)
INSTALL(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/smashVHLLEhybrid.sh ${CMAKE_CURRENT_BINARY_DIR}/smashPaths.sh ${CMAKE_CURRENT_BINARY_DIR}/smashAnalysis.sh DESTINATION bin)
