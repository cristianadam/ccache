# MIT License

# Copyright (c) 2017-2020 Cristian Adam

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

cmake_minimum_required(VERSION 3.12 FATAL_ERROR)

find_package(Git)
execute_process(COMMAND ${GIT_EXECUTABLE} describe
    OUTPUT_VARIABLE git_output
    ERROR_VARIABLE git_error
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
if (NOT git_error)
    string(REGEX MATCH "[0-9.]+" CCACHE_VERSION ${git_output})
else()
    set(CCACHE_VERSION 3.7.7)
endif()

file(WRITE ${CMAKE_BINARY_DIR}/version.cpp.in [=[
    extern const char CCACHE_VERSION[];
    const char CCACHE_VERSION[] = "@CCACHE_VERSION@";
]=])
configure_file(${CMAKE_BINARY_DIR}/version.cpp.in ${CMAKE_BINARY_DIR}/version.cpp @ONLY)

project(ccache VERSION ${CCACHE_VERSION} LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 11)

#
# zstd
#

option(USE_LIBZSTD_FROM_INTERNET "Download and use libzstd from the Internet instead of using the system's default [libzstd]" ON)
if (USE_LIBZSTD_FROM_INTERNET)
    set(zstd_version 1.4.4)
    set(zstd_url https://github.com/facebook/zstd/releases/download/v${zstd_version}/zstd-${zstd_version}.tar.gz)

    include(FetchContent)
    FetchContent_Declare(zstd_content
        URL ${zstd_url}
        SUBBUILD_DIR build/cmake)
    FetchContent_MakeAvailable(zstd_content)
    set(ZSTD_BUILD_SHARED OFF)
    add_subdirectory(${zstd_content_SOURCE_DIR}/build/cmake ${zstd_content_BINARY_DIR} EXCLUDE_FROM_ALL)

    add_library(ZSTD::ZSTD ALIAS libzstd_static)
    set_target_properties(libzstd_static PROPERTIES
      INTERFACE_INCLUDE_DIRECTORIES "${zstd_content_SOURCE_DIR}/lib"
    )
else()
    find_library(ZSTD_LIBRARY zstd)
    find_path(ZSTD_INCLUDE_DIR zstd.h)

    add_library(ZSTD::ZSTD UNKNOWN IMPORTED)
    set_target_properties(ZSTD::ZSTD PROPERTIES
      IMPORTED_LOCATION "${ZSTD_LIBRARY}"
      INTERFACE_INCLUDE_DIRECTORIES "${ZSTD_INCLUDE_DIR}"
    )
endif()

#
# libb2
#

option(USE_LIBB2_FROM_INTERNET "Download and use libb2 from the Internet instead of using the system's default [libb2]" ON)
if (USE_LIBB2_FROM_INTERNET)
    set(libb2_version 0.98.1)
    set(libb2_url https://github.com/BLAKE2/libb2/releases/download/v${libb2_version}/libb2-${libb2_version}.tar.gz)

    include(FetchContent)
    FetchContent_Declare(libb2_content URL ${libb2_url})
    FetchContent_MakeAvailable(libb2_content)

    file(WRITE ${libb2_content_SOURCE_DIR}/src/config.h.cmake.in [=[
      /* Define if you have the `explicit_bzero' function. */
      #cmakedefine HAVE_EXPLICIT_BZERO

      /* Define if you have the `explicit_memset' function. */
      #cmakedefine HAVE_EXPLICIT_MEMSET

      /* Define if you have the `memset' function. */
      #cmakedefine HAVE_MEMSET

      /* Define if you have the `memset_s' function. */
      #cmakedefine HAVE_MEMSET_S
      ]=])

    file(WRITE ${libb2_content_SOURCE_DIR}/src/CMakeLists.txt [=[
      project(libb2 C)

      include(CheckFunctionExists)
      foreach(func IN ITEMS
        explicit_bzero
        explicit_memset
        memset
        memset_s
        )
        string(TOUPPER ${func} func_var)
        set(func_var HAVE_${func_var})
        check_function_exists(${func} ${func_var})
      endforeach()

      configure_file(config.h.cmake.in config.h)
      set(CMAKE_INCLUDE_CURRENT_DIR ON)

      add_library(libblake2b_ref STATIC blake2b-ref.c blake2s-ref.c)
      target_compile_definitions(libblake2b_ref PRIVATE SUFFIX=_ref)

      function(add_libblake2b name suffix)
        add_library(${name} STATIC blake2b.c blake2s.c)
        target_compile_definitions(${name} PRIVATE ${suffix})
        target_compile_options(${name} PRIVATE ${ARGN})
      endfunction()

      add_libblake2b(libblake2b_sse2 SUFFIX=_sse2 -msse2)
      add_libblake2b(libblake2b_ssse3 SUFFIX=_ssse3 -msse2 -mssse3)
      add_libblake2b(libblake2b_sse41 SUFFIX=_sse41 -msse2 -mssse3 -msse4.1)
      add_libblake2b(libblake2s_avx SUFFIX=_avx -msse2 -mssse3 -msse4.1 -mavx)
      add_libblake2b(libblake2b_xop SUFFIX=_xop -msse2 -mssse3 -msse4.1 -mavx -mxop)

      add_library(libb2 STATIC blake2-dispatch.c)
      target_link_libraries(libb2
        PUBLIC
          libblake2b_ref libblake2b_sse2 libblake2b_ssse3
          libblake2b_sse41 libblake2s_avx libblake2b_xop
      )
    ]=])
    add_subdirectory(${libb2_content_SOURCE_DIR}/src ${libb2_content_BINARY_DIR} EXCLUDE_FROM_ALL)

    add_library(libb2::libb2 ALIAS libb2)
    set_target_properties(libb2 PROPERTIES
      INTERFACE_INCLUDE_DIRECTORIES "${libb2_content_SOURCE_DIR}/src"
    )
 else()
     find_library(LIBB2_LIBRARY b2)
     find_path(LIBB2_INCLUDE_DIR blake2b.h)

     add_library(libb2::libb2 UNKNOWN IMPORTED)
     set_target_properties(libb2::libb2 PROPERTIES
       IMPORTED_LOCATION "${LIBB2_LIBRARY}"
       INTERFACE_INCLUDE_DIRECTORIES "${LIBB2_INCLUDE_DIR}"
     )
endif()

#
# ccache
#

set(ccache_sources
    src/args.cpp
    src/AtomicFile.cpp src/AtomicFile.hpp
    src/CacheEntryReader.cpp src/CacheEntryReader.hpp
    src/CacheEntryWriter.cpp src/CacheEntryWriter.hpp
    src/CacheFile.cpp src/CacheFile.hpp
    src/ccache.cpp src/ccache.hpp
    src/Checksum.hpp
    src/cleanup.cpp src/cleanup.hpp
    src/compopt.cpp src/compopt.hpp
    src/compress.cpp src/compress.hpp
    src/Compression.cpp src/Compression.hpp
    src/Compressor.cpp src/Compressor.hpp
    src/Config.cpp src/Config.hpp
    src/counters.cpp src/counters.hpp
    src/Decompressor.cpp src/Decompressor.hpp
    src/Error.hpp
    src/execute.cpp
    src/exitfn.cpp
    src/File.hpp
    src/FormatNonstdStringView.hpp
    src/hash.cpp src/hash.hpp
    src/hashutil.cpp src/hashutil.hpp
    src/language.cpp src/language.hpp
    src/legacy_util.cpp
    src/lockfile.cpp
    src/macroskip.hpp
    src/manifest.cpp src/manifest.hpp
    src/NonCopyable.hpp
    src/NullCompressor.cpp src/NullCompressor.hpp
    src/NullDecompressor.cpp src/NullDecompressor.hpp
    src/ProgressBar.cpp src/ProgressBar.hpp
    src/result.cpp src/result.hpp
    src/Stat.cpp src/Stat.hpp
    src/stats.cpp
    src/StdMakeUnique.hpp
    src/system.hpp
    src/ThreadPool.hpp
    src/Util.cpp src/Util.hpp
    src/ZstdCompressor.cpp src/ZstdCompressor.hpp
    src/ZstdDecompressor.cpp src/ZstdDecompressor.hpp

    src/third_party/format.cpp
    src/third_party/getopt_long.c
    src/third_party/minitrace.c
    src/third_party/xxhash.c

    ${CMAKE_BINARY_DIR}/version.cpp
)

add_library(ccache_object_lib OBJECT ${ccache_sources})
target_include_directories(ccache_object_lib PUBLIC ${CMAKE_BINARY_DIR} src src/third_party)
target_link_libraries(ccache_object_lib PUBLIC ZSTD::ZSTD libb2::libb2)

add_executable(ccache src/main.cpp)
target_link_libraries(ccache PRIVATE ccache_object_lib)

install(TARGETS ccache DESTINATION ./)

#
# unittest
#

enable_testing()
set(unittest_files
    unittest/catch2_tests.cpp unittest/catch2_tests.hpp
    unittest/framework.cpp unittest/framework.hpp
    unittest/main.cpp
    unittest/test_args.cpp
    unittest/test_argument_processing.cpp
    unittest/test_AtomicFile.cpp
    unittest/test_Checksum.cpp
    unittest/test_compopt.cpp
    unittest/test_Compression.cpp
    unittest/test_Config.cpp
    unittest/test_counters.cpp
    unittest/test_FormatNonstdStringView.cpp
    unittest/test_hash.cpp
    unittest/test_hashutil.cpp
    unittest/test_legacy_util.cpp
    unittest/test_lockfile.cpp
    unittest/test_NullCompression.cpp
    unittest/test_Stat.cpp
    unittest/test_stats.cpp
    unittest/test_Util.cpp
    unittest/test_ZstdCompression.cpp
    unittest/util.cpp unittest/util.hpp
)

add_executable(unittest ${unittest_files})
target_link_libraries(unittest PRIVATE ccache_object_lib)

add_test(NAME unittest COMMAND unittest)

include(CheckIncludeFile)
foreach(include_file IN ITEMS
        pwd.h
        sys/mman.h
        sys/time.h
        sys/wait.h
        termios.h
    )
    string(TOUPPER ${include_file} include_var)
    string(REGEX REPLACE "[/.]" "_" include_var ${include_var})
    set(include_var HAVE_${include_var})
    check_include_file(${include_file} ${include_var})
endforeach()

include(CheckFunctionExists)
foreach(func IN ITEMS
        GetFinalPathNameByHandleW
        getopt_long
        getpwuid
        gettimeofday
        localtime_r
        mkstemp
        realpath
        strndup
        strtok_r
        unsetenv
        utimes
    )
    string(TOUPPER ${func} func_var)
    set(func_var HAVE_${func_var})
    check_function_exists(${func} ${func_var})
endforeach()

# It doesn't follow the above naming scheme
check_function_exists(__compar_fn_t HAVE_COMPAR_FN_T)

include(CheckSymbolExists)
list(APPEND CMAKE_REQUIRED_LIBRARIES ws2_32)
check_symbol_exists(gethostname winsock2.h HAVE_GETHOSTNAME)
list(REMOVE_ITEM CMAKE_REQUIRED_LIBRARIES ws2_32)

include(CheckTypeSize)
check_type_size("long long" HAVE_LONG_LONG)

if(WIN32)
    set(_WIN32_WINNT 0x0600)
    target_link_libraries(ccache PRIVATE ws2_32)
    target_link_libraries(unittest PRIVATE ws2_32)

    if (CMAKE_CXX_COMPILER_ID MATCHES "GNU")
        target_link_libraries (ccache PRIVATE -static gcc stdc++ winpthread -dynamic)
    else()
        target_link_libraries (ccache PRIVATE -static c++ -dynamic)
    endif()
endif()

set(_GNU_SOURCE 1)
set(_POSIX_C_SOURCE 1)

if (CMAKE_SYSTEM MATCHES "Darwin")
  set(_DARWIN_C_SOURCE 1)
endif()

configure_file(config.h.cmake.in ${CMAKE_BINARY_DIR}/config.h)
