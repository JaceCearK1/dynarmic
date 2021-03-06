cmake_minimum_required(VERSION 3.2)
project(dynarmic CXX)

# Determine if we're built as a subproject (using add_subdirectory)
# or if this is the master project.
set(MASTER_PROJECT OFF)
if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
  set(MASTER_PROJECT ON)
endif()

# Dynarmic project options
option(DYNARMIC_USE_SYSTEM_BOOST "Use the system boost libraries" ON)
option(DYNARMIC_USE_LLVM "Support disassembly of jitted x86_64 code using LLVM" OFF)
option(DYNARMIC_TESTS "Build tests" ${MASTER_PROJECT})
option(DYNARMIC_WARNINGS_AS_ERRORS "Warnings as errors" ${MASTER_PROJECT})

# Set hard requirements for C++
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Warn on CMake API deprecations
set(CMAKE_WARN_DEPRECATED ON)

# Disable in-source builds
set(CMAKE_DISABLE_SOURCE_CHANGES ON)
set(CMAKE_DISABLE_IN_SOURCE_BUILD ON)
if ("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")
    message(SEND_ERROR "In-source builds are not allowed.")
endif()

# Add the module directory to the list of paths
list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/CMakeModules")

# Compiler flags
if (MSVC)
    set(DYNARMIC_CXX_FLAGS
        /W4
        /w34263 # Non-virtual member function hides base class virtual function
        /w44265 # Class has virtual functions, but destructor is not virtual
        /w34456 # Declaration of 'var' hides previous local declaration
        /w34457 # Declaration of 'var' hides function parameter
        /w34458 # Declaration of 'var' hides class member
        /w34459 # Declaration of 'var' hides global definition
        /w34946 # Reinterpret-cast between related types
        /wd4592 # Symbol will be dynamically initialized (implementation limitation)
        /MP
        /Zi
        /Zo
        /EHsc
        /DNOMINMAX)

    if (DYNARMIC_WARNINGS_AS_ERRORS)
        list(APPEND DYNARMIC_CXX_FLAGS
             /WX)
    endif()

    if (CMAKE_VS_PLATFORM_TOOLSET MATCHES "LLVM-vs[0-9]+")
        list(APPEND DYNARMIC_CXX_FLAGS
             -Qunused-arguments
             -Wno-missing-braces
             -Xclang -fno-operator-names)
    endif()
else()
    set(DYNARMIC_CXX_FLAGS
        -Wall
        -Wextra
        -Wcast-qual
        -pedantic
        -pedantic-errors
        -Wfatal-errors
        -Wno-missing-braces
        -fno-operator-names)

    if (DYNARMIC_WARNINGS_AS_ERRORS)
        list(APPEND DYNARMIC_CXX_FLAGS
             -Werror)
    endif()

    if (ARCHITECTURE_x86_64)
        list(APPEND DYNARMIC_CXX_FLAGS
             -msse3)
    endif()
endif()

# Arch detection
include(DetectArchitecture)
if (MSVC)
    detect_architecture("_M_AMD64" x86_64)
    detect_architecture("_M_IX86" x86)
    detect_architecture("_M_ARM" ARM)
else()
    detect_architecture("__x86_64__" x86_64)
    detect_architecture("__i386__" x86)
    detect_architecture("__arm__" ARM)
endif()
if (NOT DEFINED ARCHITECTURE)
    set(ARCHITECTURE "GENERIC")
    set(ARCHITECTURE_GENERIC 1)
    add_definitions(-DARCHITECTURE_GENERIC=1)
endif()
message(STATUS "Target architecture: ${ARCHITECTURE}")

# Include Boost
if(DYNARMIC_USE_SYSTEM_BOOST)
    find_package(Boost 1.57.0 REQUIRED)
else()
    if(NOT Boost_INCLUDE_DIRS)
        message(FATAL_ERROR "Please provide a path to a boost installation using Boost_INCLUDE_DIRS")
    endif()
endif()
add_library(boost INTERFACE IMPORTED)
set_property(TARGET boost PROPERTY INTERFACE_INCLUDE_DIRECTORIES ${Boost_INCLUDE_DIRS})

# Enable unit-testing.
enable_testing(true)

# Include LLVM
if (DYNARMIC_USE_LLVM)
    find_package(LLVM REQUIRED CONFIG)
    include_directories(${LLVM_INCLUDE_DIRS})
    add_definitions(-DDYNARMIC_USE_LLVM ${LLVM_DEFINITIONS})
    llvm_map_components_to_libnames(llvm_libs x86desc x86disassembler)
endif()

# Pull in externals CMakeLists for libs where available
add_subdirectory(externals)

# Dynarmic project files
add_subdirectory(src)
if (DYNARMIC_TESTS)
    add_subdirectory(tests)
endif()
