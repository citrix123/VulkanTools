# The name of our project is "VULKAN". CMakeLists files in this project can
# refer to the root source directory of the project as ${VULKAN_SOURCE_DIR} and
# to the root binary directory of the project as ${VULKAN_BINARY_DIR}.
cmake_minimum_required(VERSION 2.8.11)
project (VULKAN_TOOLS)

set(API_NAME "Vulkan" CACHE STRING "API name")
add_definitions(-DAPI_NAME="${API_NAME}")

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
find_package(PythonInterp 3 REQUIRED)

set (PYTHON_CMD ${PYTHON_EXECUTABLE})

# Enable cmake folders
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set(VULKANTOOLS_TARGET_FOLDER vt_cmake_targets)

option(USE_CCACHE "Use ccache" OFF)
if (USE_CCACHE)
    find_program(CCACHE_FOUND ccache)
    if(CCACHE_FOUND)
        set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
        set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
    endif(CCACHE_FOUND)
endif()

if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(FALLBACK_CONFIG_DIRS "/etc/xdg" CACHE STRING
        "Search path to use when XDG_CONFIG_DIRS is unset or empty or the current process is SUID/SGID. Default is freedesktop compliant.")
    set(FALLBACK_DATA_DIRS "/usr/local/share:/usr/share" CACHE STRING
        "Search path to use when XDG_DATA_DIRS is unset or empty or the current process is SUID/SGID. Default is freedesktop compliant.")

    include(FindPkgConfig)
    option(BUILD_WSI_XCB_SUPPORT "Build XCB WSI support" ON)
    option(BUILD_WSI_XLIB_SUPPORT "Build Xlib WSI support" ON)
    option(BUILD_WSI_WAYLAND_SUPPORT "Build Wayland WSI support" ON)
    option(BUILD_WSI_MIR_SUPPORT "Build Mir WSI support" OFF)
    set(DEMOS_WSI_SELECTION "XCB" CACHE STRING "Select WSI target for demos (XCB, XLIB, WAYLAND, MIR, DISPLAY)")
    set(VKREPLAY_WSI_SELECTION "XCB" CACHE STRING "Select WSI target for vkreplay (XCB, WAYLAND, DISPLAY)")

    # Need to turn WSI_SELECTION into boolean definitions for preprocessor directives
    if (VKREPLAY_WSI_SELECTION STREQUAL "XCB")
        add_definitions(-DVKREPLAY_USE_WSI_XCB)
    elseif (VKREPLAY_WSI_SELECTION STREQUAL "WAYLAND")
        add_definitions(-DVKREPLAY_USE_WSI_WAYLAND)
    endif()

    if (BUILD_WSI_XCB_SUPPORT)
        find_package(XCB REQUIRED)
    endif()

    if (BUILD_WSI_XLIB_SUPPORT)
        find_package(X11 REQUIRED)
    endif()

    if (BUILD_WSI_WAYLAND_SUPPORT)
        find_package(Wayland REQUIRED)
        include_directories(${WAYLAND_CLIENT_INCLUDE_DIR})
    endif()

    if(BUILD_WSI_MIR_SUPPORT)
        find_package(Mir REQUIRED)
    endif()
endif()

set (VULKAN_SDK $ENV{VULKAN_SDK})
set (VALIDATION_LAYERS_RELATIVE_LOCATION submodules/Vulkan-ValidationLayers)
set (SUBPROJECTS_DIR ${CMAKE_CURRENT_SOURCE_DIR}/submodules)

add_subdirectory(${VALIDATION_LAYERS_RELATIVE_LOCATION})
set (VALIDATION_LAYERS_ROOT_DIR ${CMAKE_CURRENT_SOURCE_DIR}/${VALIDATION_LAYERS_RELATIVE_LOCATION})

set(VULKANTOOLS_SCRIPTS_DIR ${CMAKE_CURRENT_SOURCE_DIR}/scripts)
set(VALIDATION_LAYERS_SCRIPTS_DIR ${VALIDATION_LAYERS_ROOT_DIR}/scripts)

# Header file for CMake settings
find_package(VulkanHeaders)
include_directories(${VulkanHeaders_INCLUDE_DIR} ${PROJECT_BINARY_DIR}/${VALIDATION_LAYERS_RELATIVE_LOCATION})

# The vulkan loader search is:
#     User-supplied setting of CMAKE_PREFIX_PATH
#     VULKAN_LOADER_INSTALL_DIR defined via cmake option
#     VULKAN_LOADER_INSTALL_DIR defined via environment variable
#     Default findVulkan operation if the VULKAN_SDK environment variable is defined
set(VULKAN_LOADER_INSTALL_DIR "LOADER-NOTFOUND" CACHE PATH "Absolute path to a Vulkan-Loader install directory")
if (VULKAN_LOADER_INSTALL_DIR)
    message(STATUS "VULKAN_LOADER_INSTALL_DIR specified, using find_package to locate Vulkan")
elseif(ENV{VULKAN_LOADER_INSTALL_DIR})
    message(STATUS "VULKAN_LOADER_INSTALL_DIR environment variable specified, using find_package to locate Vulkan")
endif()
set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH};${VULKAN_LOADER_INSTALL_DIR};${VULKAN_HEADERS_INSTALL_DIR};$ENV{VULKAN_LOADER_INSTALL_DIR};$ENV{VULKAN_HEADERS_INSTALL_DIR})
find_package(Vulkan)
set (LIBVK "Vulkan::Vulkan")

if (CMAKE_SYSTEM_NAME STREQUAL "Windows" OR
    CMAKE_SYSTEM_NAME STREQUAL "Linux")

    option(BUILD_TESTS "Build tests" ON)
    option(BUILD_LAYERSVT "Build layersvt" ON)
    option(BUILD_VLF "Build layer_factory" ON)
    option(BUILD_VKTRACE "Build VkTrace" ON)
    option(BUILD_VIA "Build via" ON)

    if (CMAKE_SYSTEM_NAME STREQUAL "Linux" AND NOT (BUILD_WSI_XCB_SUPPORT OR BUILD_WSI_WAYLAND_SUPPORT))
        option(BUILD_VKTRACE_REPLAY "Build VkReplay" OFF)
    else()
        option(BUILD_VKTRACE_REPLAY "Build VkReplay" ON)
    endif()

elseif (CMAKE_SYSTEM_NAME STREQUAL "Darwin")

    # Only vktrace enable for macOS
    option(BUILD_VKTRACE ON)
    option(BUILD_LOADER OFF)
    option(BUILD_TESTS OFF)
    option(BUILD_LAYERS OFF)
    option(BUILD_LAYERSVT OFF)
    option(BUILD_VLF OFF)
    option(BUILD_VKTRACEVIEWER OFF)
    option(BUILD_DEMOS OFF)
    option(BUILD_VKJSON OFF)
    option(BUILD_VIA OFF)
    option(BUILD_VKTRACE_LAYER OFF)
    option(BUILD_VKTRACE_REPLAY OFF)

endif()

find_path(JSONCPP_INCLUDE_DIR json/json.h HINTS "${SUBPROJECTS_DIR}/jsoncpp/dist"
                                                   "${SUBPROJECTS_DIR}/JsonCpp/dist"
                                                   "${SUBPROJECTS_DIR}/JsonCPP/dist"
                                                   "${SUBPROJECTS_DIR}/JSONCPP/dist"
                                                   "${CMAKE_SOURCE_DIR}/../jsoncpp/dist"
                                             DOC "Path to jsoncpp/dist/json/json.h")

find_path(JSONCPP_SOURCE_DIR jsoncpp.cpp HINTS "${SUBPROJECTS_DIR}/jsoncpp/dist"
                                                   "${SUBPROJECTS_DIR}/JsonCpp/dist"
                                                   "${SUBPROJECTS_DIR}/JsonCPP/dist"
                                                   "${SUBPROJECTS_DIR}/JSONCPP/dist"
                                                   "${CMAKE_SOURCE_DIR}/../jsoncpp/dist"
                                             DOC "Path to jsoncpp/dist/json.cpp")

find_library(JSONCPP_LIB NAMES jsoncpp HINTS ${JSONCPP_SEARCH_PATH} )

if (WIN32)
    add_library(jsoncpp         STATIC IMPORTED)
    find_library(JSONCPP_DLIB NAMES jsoncpp
                 HINTS ${JSONCPP_DEBUG_SEARCH_PATH} )
    set_target_properties(jsoncpp PROPERTIES
                         IMPORTED_LOCATION       "${JSONCPP_LIB}"
                         IMPORTED_LOCATION_DEBUG "${JSONCPP_DLIB}")
endif()

# Define macro used for building vkxml generated files
macro(run_vulkantools_vk_xml_generate dependency output)
    add_custom_command(OUTPUT ${output}
    COMMAND ${PYTHON_CMD} ${VULKANTOOLS_SCRIPTS_DIR}/vt_genvk.py -registry ${VulkanRegistry_DIR}/vk.xml -scripts ${VulkanRegistry_DIR} ${output}
    DEPENDS ${VulkanRegistry_DIR}/vk.xml ${VulkanRegistry_DIR}/generator.py ${VULKANTOOLS_SCRIPTS_DIR}/${dependency} ${VULKANTOOLS_SCRIPTS_DIR}/vt_genvk.py ${VulkanRegistry_DIR}/reg.py
    )
endmacro()

if(BUILD_TESTS)
    add_subdirectory(tests)
    add_subdirectory(${SUBPROJECTS_DIR}/Vulkan-Tools)
endif()

if(BUILD_VLF)
    add_subdirectory(layer_factory)
endif()

if(BUILD_LAYERSVT)
    add_subdirectory(layersvt)
endif()

if(BUILD_VKTRACE)
    add_subdirectory(vktrace)
endif()

if(BUILD_VIA)
    add_subdirectory(via)
endif()
