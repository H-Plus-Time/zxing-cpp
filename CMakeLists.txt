#
# CMake listfile to specify the build process, see:
# http://www.cmake.org/cmake/help/documentation.html
#
project(zxing)
cmake_minimum_required(VERSION 2.8.0)
message(STATUS ${WASM_CXXFLAGS})
message(STATUS ${EXPORTED_FUNCTIONS})
set(CMAKE_LIBRARY_PATH /opt/local/lib ${CMAKE_LIBRARY_PATH})

# Check for polluted source tree.
if(EXISTS ${CMAKE_SOURCE_DIR}/CMakeCache.txt OR
    EXISTS ${CMAKE_SOURCE_DIR}/CMakeFiles)
    message(FATAL_ERROR
        "Source directory is polluted:"
        "\n  * remove CMakeCache.txt"
        "\n  * remove CMakeFiles directory")
endif()

# Suppress in-source builds.
if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
    message(FATAL_ERROR
        "CMake generation is not allowed within the source directory:"
        "\n  * mkdir build"
        "\n  * cd build"
        "\n  * Unix-like: cmake -G \"Unix Makefiles\" .."
        "\n  * Windows: cmake -G \"Visual Studio 10\" ..")
endif()

# Adjust CMake's module path.
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/)

# Suppress MSVC CRT warnings.
if(MSVC)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
    add_definitions(/Za)
endif()

if (EMSCRIPTEN)
    set(CMAKE_AR "emcc")
    set(CMAKE_STATIC_LIBRARY_SUFFIX ".bc")
    set(CMAKE_C_CREATE_STATIC_LIBRARY "<CMAKE_AR> -o <TARGET> <LINK_FLAGS> <OBJECTS>")
    set(CMAKE_CXX_CREATE_STATIC_LIBRARY "<CMAKE_AR> -o <TARGET> <LINK_FLAGS> <OBJECTS>")
endif()
if(WASM_CXXFLAGS)
    message(STATUS ${WASM_CXXFLAGS})
endif()
if(WASM_CXXFLAGS)
    set(CMAKE_CXX_FLAGS ${WASM_CXXFLAGS})
else()
    set(CMAKE_CXX_FLAGS "-std=c++11 --bind -s RESERVED_FUNCTION_POINTERS=20 -s WASM=1 -O3 -s MODULARIZE=1 -s EXPORT_NAME=\"'ZXing'\" -s DISABLE_EXCEPTION_CATCHING=0 -s EXPORTED_FUNCTIONS=\"['_decode_qr','_decode_qr_multi','_decode_any','_decode_multi']\"")
endif()

# Add libzxing library.
file(GLOB_RECURSE LIBZXING_FILES
    "./core/src/*.cpp"
    "./core/src/*.h"
    "./core/src/*.cc"
    "./core/src/*.hh"
)
include_directories("./core/src/")
add_library(libzxing STATIC ${LIBZXING_FILES})
set_target_properties(libzxing PROPERTIES PREFIX "")
add_definitions(-DNO_ICONV=1)

# Add cli executable.
file(GLOB_RECURSE ZXING_FILES
    "./emscripten/zxing.js.cpp"
)
add_executable(zxing ${ZXING_FILES})
target_link_libraries(zxing libzxing)

# Add testrunner executable.
find_package(CPPUNIT)
if(CPPUNIT_FOUND)
    file(GLOB_RECURSE TESTRUNNER_FILES
        "./core/tests/src/*.cpp"
        "./core/tests/src/*.h"
    )
    add_executable(testrunner ${TESTRUNNER_FILES})
    include_directories(${CPPUNIT_INCLUDE_DIR})
    target_link_libraries(testrunner libzxing  ${CPPUNIT_LIBRARIES})
else()
    message(STATUS "Not building testrunner, because CppUnit is missing")
endif()
