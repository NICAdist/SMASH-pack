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
