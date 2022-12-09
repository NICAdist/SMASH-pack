# The name of our project
project(smash-analysis NONE C)

cmake_minimum_required(VERSION 3.20)

# Fail if cmake is called in the source directory
if(CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR)
   message(FATAL_ERROR "You don't want to configure in the source directory!")
endif()

## Tell cmake where to find our modules
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")


# smash executable:
# $SMASH_PACK_ROOT/bin/smash

# Decide whether to plot experimental data as well. Note that in this case,
# the path to the experimental_data directory needs to be passed
option(WITH_EXP_DATA "Plot experimental data points as well." OFF)

if(EXP_DATA)
  message(STATUS "Found experimental data: ${EXP_DATA}")
  message(STATUS "Comparison to experimental measurements will be plotted.")
  option(WITH_EXP_DATA ON)
endif(EXP_DATA)

# 1) elementary tests
add_subdirectory(test/cross_sections)
add_subdirectory(test/angular_distributions)

# 2) box tests
add_subdirectory(test/detailed_balance)
add_subdirectory(test/elastic_box)
add_subdirectory(test/densities)

# 3) collider tests
add_subdirectory(test/FOPI_pions)
add_subdirectory(test/dileptons)
add_subdirectory(test/energy_scan)

if(SAMPLED_LISTS)
  # 4) afterburner test
  add_subdirectory(test/afterburner)
else()
  message(STATUS "No path for download path for sampled particle lists was given, afterburner target not available. Specify via \
cmake .. -DSAMPLED_LISTS=/link/to/lists")
endif(SAMPLED_LISTS)
