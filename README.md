# SMASH-pack
This is a repack of SMASH software bundle. Its sole purpose is to change the way how SMASH is built and provide scripts that create makefiles outside the cmake build directory.
Namely:
1. all 3rd packages have to be provided as external dependencies (same way as they would be
provided in ALICE's project aliBuild)
1. packages are built together from the same code-base (SMASH, SMASH-hadron-sampler, SMASH-analysis, SMASH-vHLLE-hybrid)
1. scripts from SMASH-vHLLE-hybrid are rewritten to be normal scripts callable without need of using CMake

For source files and manuals see original sources. Namely:
* SMASH - https://github.com/smash-transport/smash.git
* SMASH-analysis - https://github.com/smash-transport/smash-analysis.git
* SMASH-hadron-sampler - https://github.com/smash-transport/smash-hadron-sampler.git
* SMASH-vHLLE-hybrid - https://github.com/smash-transport/smash-vhlle-hybrid.git

The sources are copyied from original repositories and not linked. Therefore this repository is updated only manually if major change to original packages occurs (new tag is created). Each branch represents one given combination of packages. Since this repository is (mainly) for the nicadist project (https://git.jinr.ru/nica/nicadist/-/blob/master/smash.sh) no checks on exotic packages/versions are done and CMake expects all dependencies are in sufficient versions.

# This branch contains following versions of packages
| Package | Version |
|---|---|
|SMASH|v2.2.1|
|SMASH-analysis|SMASH-2.2ana|
|SMASH-hadron-sampler|SMASH-hadron-sampler-1.1|
|SMASH-vhlle-hybrid|master from 2022-24-10 (af49b95)|

# Changes compared to the original sources
## CMake smash, hadron-sampler
* removed check of cxx standard (we expect it to be at least c++17)
* removed check of Endian (we target mainly x86-64 which uses little-endian)
* removed -march=native flag (flags are set from external environment)
* removed EXECUTABLE_OUTPUT_PATH (we use global CMAKE_INSTALL_PREFIX)
* moved sources from src to src/SMASH, src/hadron-sampler
* BUILD_TESTING switched by default to NO
* removed doc directory (for documentation refer to the original project)
* added python tests from analysis and hybrid
* renamed sampler to hadron_sampler
## CMake smash-analysis
* created directory share/smashAnalysis containing (original) python_scripts/, test/, and CMakeLists.txt.
  These are copied to SMASH_PACK_ROOT/share/ during install phase
* moved .sh files to directory scripts. On install they are put into /bin
* removed gitstats
* search for Python (modules) moved to the main CMake. Analytics EXPECTS that modules are OK
* removed search for smash. It is expected to be in $SMASH_ROOT_PATH/bin/smash
* removed include(FloatMath) and replaced by given function in files that needed it (2 of 3 requesting)
* replaced ${CMAKE_BINARY_DIR}/smash by ${SMASH_PACK_ROOT}/bin/smash
* in python_scripts/version.py replaced check for analysis version by value obtained via command
  'git describe' obtained in SMASH-2.2ana branch of analysis (we are not inside git when running this)

## CMake smashVHLLEhybrid
* created directory share/smashVHLLEhyubrid containing (original) configs/, python_scripts/, and CMakeLists.txt. These are copied to SMASH_PACK_ROOT/share/ during install phase
* search for Python (modules) moved to the main CMake. Analytics EXPECTS that modules are OK
* removed search for smash, hlle_visc, and sampler. hlle_visc is expected to be inside ${VHLLE_ROOT}/bin, smash inside ${SMASH_PACK_ROOT}/bin. sampler was renamed to hadron_sampler and also resides inside ${SMASH_PACK_ROOT}/bin.
* removed search for and copying of eos as they are not used anywhere. If one needs eos, than ${VHLLE_ROOT}/data/eos should be used to avoid unnecessary copying of large amount of data
* removed test of SMASH_ANALYSIS presence since we know, we have it
* removed all connected to FLOW as it is not clear what package it is and how to install/use it
* removed calls to functions. They are now user-provided via file and parameter SMASH_ANALYSES
* removed hard-coded constants. User now can either alter individual constants via -Dconstant_name or set all (subset) of constants via file and parameter SMASH_SETTINGS
* added new folder "Progress" to results for tracking work of program and recreated dependencies in the program so that commands are called one-by one for each 'i' in cycle (allowing for make -j)
* added -configDir path variable to allow for (vHLLE) eos directory to be placed at arbitrary place
