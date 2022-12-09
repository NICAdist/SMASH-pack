# Collect all the cpp files
set(SAMPLER_SRC
   gen.cpp
   main.cpp
   params.cpp
   tree.cpp
   )

add_executable(hadron_sampler ${SAMPLER_SRC})
include_directories(SYSTEM include ${CMAKE_SOURCE_DIR}/src/smash/include)
target_link_libraries(hadron_sampler smash_shared ${ROOT_LIBRARIES} ${SMASH_LIBRARIES})
INSTALL(TARGETS hadron_sampler RUNTIME DESTINATION bin)
