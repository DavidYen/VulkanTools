cmake_minimum_required(VERSION 2.8)

if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
   if (BUILD_WSI_XCB_SUPPORT)
       set(ENV{VULKAN_WSI} "Xcb")
   elseif (BUILD_WSI_XLIB_SUPPORT)
       set(ENV{VULKAN_WSI} "Xlib")
   elseif (BUILD_WSI_WAYLAND_SUPPORT)
       set(ENV{VULKAN_WSI} "Wayland")
   else()
       # Mir WSI Case
       set(ENV{VULKAN_WSI} "Mir")
   endif()
endif()

project(VkLayer_vktrace_layer)

include("${SRC_DIR}/build_options.cmake")

file(MAKE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/codegen)
execute_process(COMMAND ${PYTHON_EXECUTABLE} ${VKTRACE_VULKAN_DIR}/vktrace_generate.py ${DisplayServer} vktrace-trace-h     vk_core OUTPUT_FILE ${CMAKE_CURRENT_SOURCE_DIR}/codegen/vktrace_vk_vk.h)
execute_process(COMMAND ${PYTHON_EXECUTABLE} ${VKTRACE_VULKAN_DIR}/vktrace_generate.py ${DisplayServer} vktrace-trace-c     vk_core OUTPUT_FILE ${CMAKE_CURRENT_SOURCE_DIR}/codegen/vktrace_vk_vk.cpp)
execute_process(COMMAND ${PYTHON_EXECUTABLE} ${VKTRACE_VULKAN_DIR}/vktrace_generate.py ${DisplayServer} vktrace-ext-trace-h vk_lunarg_debug_marker OUTPUT_FILE ${CMAKE_CURRENT_SOURCE_DIR}/codegen/vktrace_vk_vk_lunarg_debug_marker.h)
execute_process(COMMAND ${PYTHON_EXECUTABLE} ${VKTRACE_VULKAN_DIR}/vktrace_generate.py ${DisplayServer} vktrace-ext-trace-c vk_lunarg_debug_marker OUTPUT_FILE ${CMAKE_CURRENT_SOURCE_DIR}/codegen/vktrace_vk_vk_lunarg_debug_marker.cpp)

if (WIN32)
    # Put VkLayer_vktrace_layer.dll in the same directory as vktrace.exe
    # so that vktrace.exe can find VkLayer_vktrace_layer.dll.
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/../../../layers/)
endif()

set (CODEGEN_UTILS_DIR  ${CMAKE_CURRENT_SOURCE_DIR}/../vktrace_extensions/vktracevulkan/vulkan/codegen_utils)
set(CODEGEN_VKTRACE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../vktrace_extensions/vktracevulkan/codegen_vktrace_utils")

set(SRC_LIST
    ${SRC_LIST}
    vktrace_lib.c
    vktrace_lib_trace.cpp
    vktrace_vk_exts.cpp
    codegen/vktrace_vk_vk.cpp
    codegen/vktrace_vk_vk_lunarg_debug_marker.cpp
    ${CODEGEN_UTILS_DIR}/vk_struct_size_helper.c
    ${CODEGEN_UTILS_DIR}/vk_lunarg_debug_marker_struct_size_helper.c
)

set_source_files_properties( ${SRC_LIST} PROPERTIES LANGUAGE CXX)

set (HDR_LIST
    vktrace_lib_helpers.h
    vktrace_vk_exts.h
    vk_dispatch_table_helper.h
    codegen/vktrace_vk_vk.h
    codegen/vktrace_vk_vk_lunarg_debug_marker.h
    ${CODEGEN_VKTRACE_DIR}/vktrace_vk_packet_id.h
    ${CODEGEN_VKTRACE_DIR}/vktrace_vk_vk_packets.h
    ${CODEGEN_VKTRACE_DIR}/vktrace_vk_vk_lunarg_debug_marker_packets.h
    ${CODEGEN_UTILS_DIR}/vk_struct_size_helper.h
    ${CODEGEN_UTILS_DIR}/vk_lunarg_debug_marker_struct_size_helper.h
)

add_custom_command(OUTPUT vk_dispatch_table_helper.h
	COMMAND ${PYTHON_CMD} ${CMAKE_SOURCE_DIR}/vk-generate.py ${DisplayServer} dispatch-table-ops layer > vk_dispatch_table_helper.h
	DEPENDS ${CMAKE_SOURCE_DIR}/vk-generate.py ${CMAKE_SOURCE_DIR}/vulkan.py)
add_custom_target(generate_vktrace_layer_helpers DEPENDS
	vk_dispatch_table_helper.h)

include_directories(
    codegen
    ${SRC_DIR}/vktrace_common
    ${SRC_DIR}/vktrace_trace
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CODEGEN_VKTRACE_DIR}
    ${VKTRACE_VULKAN_INCLUDE_DIR}
    ${CODEGEN_UTILS_DIR}
    ${CMAKE_CURRENT_BINARY_DIR}/../../../layers
)
# copy/link layer json file into build/layers directory
if (NOT WIN32)
    # extra setup for out-of-tree builds
    if (NOT (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR))
        add_custom_target(vktrace_layer-json ALL
            COMMAND ln -sf ${CMAKE_CURRENT_SOURCE_DIR}/linux/VkLayer_vktrace_layer.json ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}../layers/
            VERBATIM
            )
    endif()
else()
    if (NOT (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR))
        FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/windows/VkLayer_vktrace_layer.json src_json)
        FILE(TO_NATIVE_PATH ${PROJECT_BINARY_DIR}/../../../layers/$<CONFIGURATION>/VkLayer_vktrace_layer.json dst_json)
        add_custom_target(vktrace_layer-json ALL
            COMMAND copy ${src_json} ${dst_json}
            VERBATIM
            )
    endif()
endif()

add_library(${PROJECT_NAME} SHARED ${SRC_LIST} ${HDR_LIST})

add_dependencies(${PROJECT_NAME} generate_vktrace_layer_helpers)

if (${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
    set(OS_TRACER_LIBS
        -shared
        -ldl
    )
endif()

if (${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")
    set(OS_TRACER_LIBS)
endif()

target_link_libraries(${PROJECT_NAME}
    vktrace_common
    ${VKTRACE_VULKAN_LIB}
    ${OS_TRACER_LIBS}
)

build_options_finalize()

set_target_properties(VkLayer_vktrace_layer PROPERTIES LINKER_LANGUAGE C)
