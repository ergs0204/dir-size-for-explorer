cmake_minimum_required(VERSION 3.24)
project(DirSizeForExplorer VERSION 1.1.1 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Match the x64-windows-static dependencies; ship self-contained binaries.
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
if(MSVC)
    add_compile_options(/utf-8)
endif()

enable_testing()

# Only support x64 builds
if(NOT CMAKE_SIZEOF_VOID_P EQUAL 8)
    message(FATAL_ERROR "Only 64-bit builds are supported")
endif()

# Find SQLite via vcpkg
find_package(unofficial-sqlite3 CONFIG REQUIRED)

# Windows SDK libraries used across multiple targets
set(DIRSIZE_WIN_LIBS
    advapi32
    shell32
    shlwapi
    ole32
    oleaut32
    propsys
    uuid
)

add_subdirectory(common)
add_subdirectory(shell-ext)
add_subdirectory(service)
add_subdirectory(tray)
add_subdirectory(tests)
