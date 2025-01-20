# Top-level CMakeLists.txt

cmake_minimum_required(VERSION 3.14)
project(SimaiBuild)

# Set the root directory and other project-related directories
set(ROOT_DIR "${CMAKE_SOURCE_DIR}")
set(NS3_DIR "${ROOT_DIR}/ns-3-alibabacloud")
set(SIMAI_DIR "${ROOT_DIR}/astra-sim-alibabacloud")
set(SOURCE_NS3_BIN_DIR "${SIMAI_DIR}/extern/network_backend/ns3-interface/simulation/build/scratch/ns3.36.1-AstraSimNetwork-debug")
set(SOURCE_ANA_BIN_DIR "${SIMAI_DIR}/build/simai_analytical/build/simai_analytical/SimAI_analytical")
set(SOURCE_PHY_BIN_DIR "${SIMAI_DIR}/build/simai_phy/build/simai_phynet/SimAI_phynet")
set(TARGET_BIN_DIR "${CMAKE_SOURCE_DIR}/bin")
set(RESULTS_DIR "${ROOT_DIR}/results")

# Add an option for each of the build modes (ns3, phy, analytical)
option(BUILD_NS3 "Build NS3 simulator" OFF)
option(BUILD_PHY "Build PHY network simulator" OFF)
option(BUILD_ANA "Build Analytical simulator" OFF)

# Helper function to create symlink to the target binary directory
function(create_symlink src dest)
    if(EXISTS ${dest})
        message(STATUS "Removing existing symlink at ${dest}")
        file(REMOVE ${dest})
    endif()
    execute_process(COMMAND ln -s ${src} ${dest} RESULT_VARIABLE result)
    if(result)
        message(FATAL_ERROR "Failed to create symlink from ${src} to ${dest}")
    else()
        message(STATUS "Created symlink from ${src} to ${dest}")
    endif()
endfunction()

# Helper function for clean-up
function(cleanup_build mode)
    if(mode STREQUAL "ns3")
        file(REMOVE_RECURSE ${SIMAI_DIR}/extern/network_backend/ns3-interface)
        execute_process(COMMAND /bin/bash ${SIMAI_DIR}/build.sh -lr ns3)
    elseif(mode STREQUAL "phy")
        execute_process(COMMAND /bin/bash ${SIMAI_DIR}/build.sh -lr phy)
    elseif(mode STREQUAL "analytical")
        execute_process(COMMAND /bin/bash ${SIMAI_DIR}/build.sh -lr analytical)
    endif()
endfunction()

# If the BUILD_NS3 option is ON, perform the ns3 build
if(BUILD_NS3)
    message(STATUS "Building NS3 simulator...")
    file(MAKE_DIRECTORY ${TARGET_BIN_DIR})
    file(REMOVE ${TARGET_BIN_DIR}/SimAI_simulator)
    file(REMOVE_RECURSE ${SIMAI_DIR}/extern/network_backend/ns3-interface)
    file(COPY ${NS3_DIR}/. DESTINATION ${SIMAI_DIR}/extern/network_backend/ns3-interface)
    execute_process(COMMAND /bin/bash ${SIMAI_DIR}/build.sh -lr ns3)
    execute_process(COMMAND /bin/bash ${SIMAI_DIR}/build.sh -c ns3)
    create_symlink(${SOURCE_NS3_BIN_DIR} ${TARGET_BIN_DIR}/SimAI_simulator)
endif()

# If the BUILD_PHY option is ON, perform the PHY build
if(BUILD_PHY)
    message(STATUS "Building PHY network simulator...")
    file(MAKE_DIRECTORY ${TARGET_BIN_DIR})
    file(REMOVE ${TARGET_BIN_DIR}/SimAI_phynet)
    execute_process(COMMAND /bin/bash ${SIMAI_DIR}/build.sh -lr phy)
    execute_process(COMMAND /bin/bash ${SIMAI_DIR}/build.sh -c phy)
    create_symlink(${SOURCE_PHY_BIN_DIR} ${TARGET_BIN_DIR}/SimAI_phynet)
endif()

# If the BUILD_ANA option is ON, perform the analytical build
if(BUILD_ANA)
    message(STATUS "Building Analytical simulator...")
    file(MAKE_DIRECTORY ${TARGET_BIN_DIR})
    file(MAKE_DIRECTORY ${RESULTS_DIR})
    file(REMOVE ${TARGET_BIN_DIR}/SimAI_analytical)
    execute_process(COMMAND /bin/bash ${SIMAI_DIR}/build.sh -lr analytical)
    execute_process(COMMAND /bin/bash ${SIMAI_DIR}/build.sh -c analytical)
    create_symlink(${SOURCE_ANA_BIN_DIR} ${TARGET_BIN_DIR}/SimAI_analytical)
endif()

# Clean-up logic
if(BUILD_NS3)
    add_custom_target(clean_ns3
            COMMAND ${CMAKE_COMMAND} -E remove -f ${TARGET_BIN_DIR}/SimAI_simulator
            COMMAND ${CMAKE_COMMAND} -E remove_directory ${SIMAI_DIR}/extern/network_backend/ns3-interface
            COMMAND /bin/bash ${SIMAI_DIR}/build.sh -lr ns3
            COMMENT "Cleaning NS3 build"
    )
endif()

if(BUILD_PHY)
    add_custom_target(clean_phy
            COMMAND ${CMAKE_COMMAND} -E remove -f ${TARGET_BIN_DIR}/SimAI_phynet
            COMMAND /bin/bash ${SIMAI_DIR}/build.sh -lr phy
            COMMENT "Cleaning PHY build"
    )
endif()

if(BUILD_ANA)
    add_custom_target(clean_analytical
            COMMAND ${CMAKE_COMMAND} -E remove -f ${TARGET_BIN_DIR}/SimAI_analytical
            COMMAND /bin/bash ${SIMAI_DIR}/build.sh -lr analytical
            COMMENT "Cleaning Analytical build"
    )
endif()

# Help message
message(STATUS "CMake build options:")
message(STATUS "-DBUILD_NS3=ON to build NS3 simulator")
message(STATUS "-DBUILD_PHY=ON to build PHY simulator")
message(STATUS "-DBUILD_ANA=ON to build Analytical simulator")