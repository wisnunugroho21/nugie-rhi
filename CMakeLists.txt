#
# Copyright (c) 2014-2021, NVIDIA CORPORATION. All rights reserved.
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and/or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.  IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
# DEALINGS IN THE SOFTWARE.


cmake_minimum_required(VERSION 3.10)

project(rhi VERSION 1.0.0)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(CMakeDependentOption)

option(RHI_BUILD_SHARED "Build RHI as a shared library (DLL or .so)" OFF)
option(RHI_INSTALL "Generate install rules for RHI" ON)
cmake_dependent_option(RHI_INSTALL_EXPORTS "Install CMake exports" OFF "RHI_INSTALL" OFF)

option(RHI_WITH_VALIDATION "Build RHI the validation layer" ON)
option(RHI_WITH_VULKAN "Build the RHI Vulkan backend" ON)
option(RHI_WITH_RTXMU "Use RTXMU for acceleration structure management" OFF)
option(RHI_WITH_AFTERMATH "Include Aftermath support (requires NSight Aftermath SDK)" OFF)

cmake_dependent_option(RHI_WITH_NVAPI "Include NVAPI support (requires NVAPI SDK)" OFF "WIN32" OFF)
cmake_dependent_option(RHI_WITH_DX11 "Build the RHI D3D11 backend" ON "WIN32" OFF)
cmake_dependent_option(RHI_WITH_DX12 "Build the RHI D3D12 backend" ON "WIN32" OFF)

if (RHI_BUILD_SHARED)
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
endif()

if (MSVC)
    string(REGEX REPLACE "/W[0-4]\\s*" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
    string(APPEND CMAKE_CXX_FLAGS " /W4 /MP")
endif()


set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

if (RHI_WITH_VULKAN AND NOT TARGET Vulkan-Headers AND NOT TARGET Vulkan::Headers)
    add_subdirectory(libs/Vulkan-Headers)
endif()

if (RHI_WITH_RTXMU)
    if (TARGET Vulkan-Headers)
        get_target_property(RTXMU_VULKAN_INCLUDE_DIR Vulkan-Headers INTERFACE_INCLUDE_DIRECTORIES)
    elseif (TARGET Vulkan::Headers)
        get_target_property(RTXMU_VULKAN_INCLUDE_DIR Vulkan::Headers INTERFACE_INCLUDE_DIRECTORIES)
    endif()

    option(RTXMU_WITH_VULKAN "" ${RHI_WITH_VULKAN})
    option(RTXMU_WITH_D3D12 "" ${RHI_WITH_DX12})

    add_subdirectory(libs/rtxmu)
endif()

if (RHI_WITH_NVAPI AND NOT TARGET nvapi)
    find_package(NVAPI REQUIRED)

    if (NVAPI_FOUND)
        add_library(nvapi STATIC IMPORTED GLOBAL)
        target_include_directories(nvapi INTERFACE "${NVAPI_INCLUDE_DIR}")
        set_property(TARGET nvapi PROPERTY IMPORTED_LOCATION "${NVAPI_LIBRARY}")
    endif()
endif()

if (RHI_WITH_AFTERMATH AND NOT TARGET aftermath)
    find_package(Aftermath REQUIRED)

    if (AFTERMATH_FOUND)
        add_library(aftermath SHARED IMPORTED GLOBAL)
        target_include_directories(aftermath INTERFACE "${AFTERMATH_INCLUDE_DIR}")
        if (WIN32)
            set_property(TARGET aftermath PROPERTY IMPORTED_IMPLIB "${AFTERMATH_LIBRARY}")
        endif()
        set_property(TARGET aftermath PROPERTY IMPORTED_LOCATION "${AFTERMATH_RUNTIME_LIBRARY}")
        file(COPY ${AFTERMATH_RUNTIME_LIBRARY} DESTINATION ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
    else()
        message(FATAL_ERROR "RHI_WITH_AFTERMATH is enabled but cmake cannot find the Aftermath SDK in AFTERMATH_SEARCH_PATHS")
    endif()
endif()

set(include_common
    include/rhi/rhi.h
    include/rhi/utils.h
    include/rhi/common/containers.h
    include/rhi/common/misc.h
    include/rhi/common/resource.h
    include/rhi/common/aftermath.h)
set(src_common
    src/common/format-info.cpp
    src/common/misc.cpp
    src/common/state-tracking.cpp
    src/common/state-tracking.h
    src/common/utils.cpp
    src/common/aftermath.cpp)

if(MSVC)
    set(misc_common 
        tools/rhi.natvis)
else()
    set(misc_common "")
endif()

set(include_validation
    include/rhi/validation.h)
set(src_validation
    src/validation/validation-commandlist.cpp
    src/validation/validation-device.cpp
    src/validation/validation-backend.h
    src/common/sparse-bitset.h
    src/common/sparse-bitset.cpp)

set(include_d3d12
    include/rhi/d3d12.h)
set(src_d3d12
    src/common/dxgi-format.h
    src/common/dxgi-format.cpp
    src/common/versioning.h
    src/d3d12/d3d12-buffer.cpp
    src/d3d12/d3d12-commandlist.cpp
    src/d3d12/d3d12-compute.cpp
    src/d3d12/d3d12-constants.cpp
    src/d3d12/d3d12-backend.h
    src/d3d12/d3d12-descriptor-heap.cpp
    src/d3d12/d3d12-device.cpp
    src/d3d12/d3d12-graphics.cpp
    src/d3d12/d3d12-meshlets.cpp
    src/d3d12/d3d12-queries.cpp
    src/d3d12/d3d12-raytracing.cpp
    src/d3d12/d3d12-resource-bindings.cpp
    src/d3d12/d3d12-shader.cpp
    src/d3d12/d3d12-state-tracking.cpp
    src/d3d12/d3d12-texture.cpp
    src/d3d12/d3d12-upload.cpp)

set(include_vk
    include/rhi/vulkan.h)
set(src_vk
    src/common/versioning.h
    src/vulkan/vulkan-allocator.cpp
    src/vulkan/vulkan-buffer.cpp
    src/vulkan/vulkan-commandlist.cpp
    src/vulkan/vulkan-compute.cpp
    src/vulkan/vulkan-constants.cpp
    src/vulkan/vulkan-device.cpp
    src/vulkan/vulkan-graphics.cpp
    src/vulkan/vulkan-meshlets.cpp
    src/vulkan/vulkan-queries.cpp
    src/vulkan/vulkan-queue.cpp
    src/vulkan/vulkan-raytracing.cpp
    src/vulkan/vulkan-resource-bindings.cpp
    src/vulkan/vulkan-shader.cpp
    src/vulkan/vulkan-staging-texture.cpp
    src/vulkan/vulkan-state-tracking.cpp
    src/vulkan/vulkan-texture.cpp
    src/vulkan/vulkan-upload.cpp
    src/vulkan/vulkan-backend.h)

# RHI interface and common implementation functions

if (RHI_BUILD_SHARED)
    add_library(rhi SHARED
        ${include_common}
        ${src_common}
        ${misc_common})
else()
    add_library(rhi STATIC
        ${include_common}
        ${src_common}
        ${misc_common})
endif()


if (RHI_BUILD_SHARED)
    target_compile_definitions(rhi PRIVATE RHI_SHARED_LIBRARY_BUILD=1)
    if(RHI_WITH_VULKAN)
        target_compile_definitions(rhi PRIVATE VULKAN_HPP_STORAGE_SHARED VULKAN_HPP_STORAGE_SHARED_EXPORT)
    endif()
    set_property(TARGET rhi PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

if (RHI_WITH_VALIDATION)
    target_sources(rhi PRIVATE
        ${include_validation}
        ${src_validation})
endif()

target_include_directories(rhi PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:$<INSTALL_PREFIX>/include>)

set_target_properties(rhi PROPERTIES FOLDER "RHI")

target_compile_definitions(rhi PRIVATE RHI_WITH_AFTERMATH=$<BOOL:${RHI_WITH_AFTERMATH}>)

# implementations

if (RHI_WITH_DX12)
    if (RHI_BUILD_SHARED)
        set(rhi_d3d12_target rhi)

        target_sources(${rhi_d3d12_target} PRIVATE
            ${include_d3d12}
            ${src_d3d12})
    else()
        set(rhi_d3d12_target rhi_d3d12)

        add_library(${rhi_d3d12_target} STATIC
            ${include_d3d12}
            ${src_d3d12})

        set_target_properties(${rhi_d3d12_target} PROPERTIES FOLDER "RHI")
        target_include_directories(${rhi_d3d12_target} PRIVATE include)
    endif()

    if (RHI_WITH_RTXMU)
        target_compile_definitions(${rhi_d3d12_target} PUBLIC RHI_WITH_RTXMU=1)
        target_link_libraries(${rhi_d3d12_target} PUBLIC rtxmu)
    endif()

    target_link_libraries(${rhi_d3d12_target} PUBLIC d3d12 dxguid)

    if (RHI_WITH_NVAPI)
        target_link_libraries(${rhi_d3d12_target} PUBLIC nvapi)
    endif()
    target_compile_definitions(${rhi_d3d12_target} PRIVATE RHI_D3D12_WITH_NVAPI=$<BOOL:${RHI_WITH_NVAPI}>)

    if (RHI_WITH_AFTERMATH)
        target_link_libraries(${rhi_d3d12_target} PUBLIC aftermath)
    endif()
    target_compile_definitions(${rhi_d3d12_target} PRIVATE RHI_WITH_AFTERMATH=$<BOOL:${RHI_WITH_AFTERMATH}>)
endif()

if (RHI_WITH_VULKAN)
    if (RHI_BUILD_SHARED)
        set(rhi_vulkan_target rhi)

        target_sources(${rhi_vulkan_target} PRIVATE
            ${include_vk}
            ${src_vk})
    else()
        set(rhi_vulkan_target rhi_vk)

        add_library(${rhi_vulkan_target} STATIC
            ${include_vk}
            ${src_vk})

        set_target_properties(${rhi_vulkan_target} PROPERTIES FOLDER "RHI")
        target_include_directories(${rhi_vulkan_target} PRIVATE include)
    endif()
    
    if(WIN32)
        target_compile_definitions(${rhi_vulkan_target} PUBLIC VK_USE_PLATFORM_WIN32_KHR)
        target_compile_definitions(${rhi_vulkan_target} PRIVATE NOMINMAX)
    endif()

    if (RHI_WITH_RTXMU)
        target_compile_definitions(${rhi_vulkan_target} PUBLIC RHI_WITH_RTXMU=1)
        target_link_libraries(${rhi_vulkan_target} PUBLIC rtxmu)
    endif()

    if (TARGET Vulkan-Headers)
        target_link_libraries(${rhi_vulkan_target} PRIVATE Vulkan-Headers)
    elseif (TARGET Vulkan::Headers)
        target_link_libraries(${rhi_vulkan_target} PRIVATE Vulkan::Headers)
    endif()

    if (RHI_WITH_AFTERMATH)
        target_link_libraries(${rhi_vulkan_target} PUBLIC aftermath)
    endif()
    target_compile_definitions(${rhi_vulkan_target} PRIVATE RHI_WITH_AFTERMATH=$<BOOL:${RHI_WITH_AFTERMATH}>)
endif()


if (RHI_INSTALL)
    install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/rhi
        DESTINATION ${CMAKE_INSTALL_PREFIX}/include)

    install(TARGETS rhi
        EXPORT "rhiTargets"
        RUNTIME DESTINATION bin
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib)

    if (NOT RHI_BUILD_SHARED)
        if (RHI_WITH_DX12)
            install(TARGETS ${rhi_d3d12_target} DESTINATION "lib" EXPORT "rhiTargets")
        endif()

        if (RHI_WITH_VULKAN)
            install(TARGETS ${rhi_vulkan_target} DESTINATION "lib" EXPORT "rhiTargets")
        endif()
    endif()

    if (RHI_INSTALL_EXPORTS)
        set(rhi_CONFIG_PATH "lib/cmake/rhi")

        include(CMakePackageConfigHelpers)

        configure_package_config_file(
            src/rhiConfig.cmake.in
            src/rhiConfig.cmake
            INSTALL_DESTINATION "${rhi_CONFIG_PATH}"
            NO_CHECK_REQUIRED_COMPONENTS_MACRO)

        write_basic_package_version_file(
            src/rhiConfigVersion.cmake 
            VERSION ${rhi_VERSION} 
            COMPATIBILITY ExactVersion)

        install(FILES
            "${rhi_BINARY_DIR}/src/rhiConfig.cmake"
            "${rhi_BINARY_DIR}/src/rhiConfigVersion.cmake"
            DESTINATION "${rhi_CONFIG_PATH}")

        install(EXPORT "rhiTargets" 
            FILE "rhiTargets.cmake"
            EXPORT_LINK_INTERFACE_LIBRARIES
            DESTINATION "${rhi_CONFIG_PATH}")
    endif()
endif()