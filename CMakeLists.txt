cmake_minimum_required(VERSION 2.8.12.1)


set(CMAKE_CONFIGURATION_TYPES Debug Release)

project(windows-shadow)

# Use folders in the resulting project files.
set_property(GLOBAL PROPERTY OS_FOLDERS ON)

# 下载好 cef 把 Debug Release Debug include libcef_dll Resources 复制到 libcef


# 设置cmake
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
# 设置cef路径
set(CEF_ROOT "${PROJECT_SOURCE_DIR}/libcef")


find_package(CEF REQUIRED)

add_subdirectory(${CEF_LIBCEF_DLL_WRAPPER_PATH} libcef_dll_wrapper)

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/shadow-core")
	add_subdirectory("shadow-core")
endif()

# 打印cef 的设置信息
PRINT_CEF_CONFIG()
