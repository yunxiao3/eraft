# MIT License

# Copyright (c) 2021 Colin

# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:

# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.

# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

cmake_minimum_required(VERSION 3.8)
set(This ectdraftcpp)
project(${This} C CXX)

set(CMAKE_C_STANDARD 99)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

enable_testing()

get_directory_property(ParentDirectory PARENT_DIRECTORY)
if(ParentDirectory STREQUAL "")
    # Define option for linking with runtime statically
    if(MSVC)
        option(RUNTIME_STATIC_LINKAGE "Link with runtime library statically" OFF)
    endif(MSVC)

    if(MSVC AND RUNTIME_STATIC_LINKAGE)
        foreach(flag
            CMAKE_C_FLAGS
            CMAKE_CXX_FLAGS
            CMAKE_C_FLAGS_DEBUG
            CMAKE_CXX_FLAGS_DEBUG
            CMAKE_C_FLAGS_RELEASE
            CMAKE_CXX_FLAGS_RELEASE
            CMAKE_C_FLAGS_MINSIZEREL
            CMAKE_CXX_FLAGS_MINSIZEREL
            CMAKE_C_FLAGS_RELWITHDEBINFO
            CMAKE_CXX_FLAGS_RELWITHDEBINFO
        )
            if(${flag} MATCHES "/MD")
                string(REGEX REPLACE "/MD" "/MI" ${flag} "${${flag}}")
            endif(${flag} MATCHES "/MD")
        endforeach(flag)
    endif(MSVC AND RUNTIME_STATIC_LINKAGE)

    # Keep Google Test's linkage of the runtime in sync our own
    if(RUNTIME_STATIC_LINKAGE)
        set(gtest_force_shared_crt OFF CACHE BOOL "Use shared (DLL) run-time lib even when Google Test is built as static lib." FORCE)
    else(RUNTIME_STATIC_LINKAGE)
        set(gtest_force_shared_crt ON CACHE BOOL "Use shared (DLL) run-time lib even when Google Test is built as static lib." FORCE)
    endif(RUNTIME_STATIC_LINKAGE)

    # Orgnize application, libraries, etc. in floders in the IDE.
    set_property(GLOBAL PROPERTY USE_FLODERS ON)

    # Ensure proper loading of side-loaded shared libraries on all target
    set(CMAKE_SKIP_BUILD_RPATH FALSE)
    set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
    if(APPLE)
        set(CMAKE_INSTALL_RPATH "@loader_path")
    elseif(UNIX)
        set(CMAKE_INSTALL_RPATH "\$ORIGIN")
    endif()
    
    # Don't add the goofy "d" postfix for debug builds of libraries.
    set(CMAKE_DEBUG_POSTFIX "")
    
    # Suppress Microsoft warnings about functions being unsafe.
    add_definitions(-D_CRT_SECURE_NO_WARNINGS -D_SCL_SECURE_NO_WARNINGS)

    # Google Test
    set(BUILD_GTEST ON CACHE BOOL "Builds the googletest subproject" FORCE)
    set(BUILD_GMOCK OFF CACHE BOOL "Builds the googlemock subproject" FORCE)

    add_subdirectory(googletest)
    set_target_properties(gtest_main gtest PROPERTIES
        FOLDER libraries
    )

    # grpc
    include_directories(grpc/include)

    # gRPC and relateds

    find_package (OpenSSL)
    message (STATUS "Using ssl=${OPENSSL_FOUND}: ${OPENSSL_INCLUDE_DIR} : ${OPENSSL_LIBRARIES}")

    find_package(Protobuf REQUIRED)
    message(STATUS "Using protobuf: ${Protobuf_VERSION} : ${Protobuf_INCLUDE_DIRS}, ${Protobuf_LIBRARIES}")

    include_directories(${PROTOBUF_INCLUDE_DIRS})

    find_package(c-ares REQUIRED)
    message(STATUS "Lib c-ares found")

    find_package(ZLIB REQUIRED)
    message(STATUS "Using ZLIB: ${ZLIB_INCLUDE_DIRS}, ${ZLIB_LIBRARIES}")

    find_package(gRPC CONFIG REQUIRED)
    message(STATUS "Using gRPC: ${gRPC_VERSION}")

    find_package(Gflags REQUIRED)
    include_directories(${GFLAGS_INCLUDE_DIRS})

    find_path(ROCKSDB_ROOT_DIR
        NAMES include/rocksdb/db.h
    )

    find_library(ROCKSDB_LIBRARIES
        NAMES rocksdb
        HINTS ${ROCKSDB_ROOT_DIR}/lib
    )

    find_path(ROCKSDB_INCLUDE_DIRS
        NAMES rocksdb/db.h
        HINTS ${ROCKSDB_ROOT_DIR}/include
    )

    include(FindPackageHandleStandardArgs)
    find_package_handle_standard_args(RocksDB DEFAULT_MSG
        ROCKSDB_LIBRARIES
        ROCKSDB_INCLUDE_DIRS
    )

    mark_as_advanced(
        ROCKSDB_ROOT_DIR
        ROCKSDB_LIBRARIES
        ROCKSDB_INCLUDE_DIRS
    )

    add_subdirectory(protocol)
    add_subdirectory(raftcore)
    add_subdirectory(kv)
    add_subdirectory(meta_server)
    add_subdirectory(cppstringplus)

endif(ParentDirectory STREQUAL "")

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
