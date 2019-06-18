# MIT License

# Copyright (c) 2017-2019 Cristian Adam

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
    set(CCACHE_VERSION 3.7.2)
endif()

file(WRITE ${CMAKE_BINARY_DIR}/version.c.in [=[
extern const char CCACHE_VERSION[];
const char CCACHE_VERSION[] = "@CCACHE_VERSION@";
]=])
configure_file(${CMAKE_BINARY_DIR}/version.c.in ${CMAKE_BINARY_DIR}/version.c @ONLY)

project(ccache VERSION ${CCACHE_VERSION} LANGUAGES C CXX)

#
# gperf
#
find_program(gperf_program gperf)
if (NOT gperf_program)
    file(WRITE ${CMAKE_BINARY_DIR}/gperf/config.h "")
    set(gperf_sources
        src/third_party/gperf/lib/getline.cc
        src/third_party/gperf/lib/hash.cc
        src/third_party/gperf/src/bool-array.cc
        src/third_party/gperf/src/hash-table.cc
        src/third_party/gperf/src/input.cc
        src/third_party/gperf/src/keyword-list.cc
        src/third_party/gperf/src/keyword.cc
        src/third_party/gperf/src/main.cc
        src/third_party/gperf/src/options.cc
        src/third_party/gperf/src/output.cc
        src/third_party/gperf/src/positions.cc
        src/third_party/gperf/src/search.cc
        src/third_party/gperf/src/version.cc
    )
    add_executable(gperf ${gperf_sources})
    target_include_directories(gperf PRIVATE src/third_party/gperf/lib ${CMAKE_BINARY_DIR}/gperf)
endif()

#
# zstd
#

option(USE_LIBZSTD_FROM_INTERNET "Download and use libzstd from the Internet instead of using the system's default [libzstd])" OFF)
if (USE_LIBZSTD_FROM_INTERNET)
    set(zstd_version 1.4.1)
    set(zstd_url https://github.com/facebook/zstd/releases/download/v${zstd_version}/zstd-${zstd_version}.tar.gz)

    include(FetchContent)
    FetchContent_Declare(zstd_content
        URL ${zstd_url}
        SUBBUILD_DIR build/cmake)
    FetchContent_MakeAvailable(zstd_content)
    set(ZSTD_BUILD_SHARED OFF)
    add_subdirectory(${zstd_content_SOURCE_DIR}/build/cmake ${zstd_content_BINARY_DIR})

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
# ccache
#

# CMake script that extracts the value of TOTAL_KEYWORDS and writes a C file
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/total_keywords.cmake [=[
    file(STRINGS ${CMAKE_CURRENT_BINARY_DIR}/${SOURCE}_lookup.c ${SOURCE}_numbers_line
        REGEX "TOTAL_KEYWORDS = [0-9]+,")
    string(REGEX MATCH "([0-9]+)" ${SOURCE}_numbers ${${SOURCE}_numbers_line})
    file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/${SOURCE}_lookup.c
        "size_t ${SOURCE}_count(void) { return ${${SOURCE}_numbers}; }\n")
]=])

# CMake script that will assign the sorted index of ITEM
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/add_sorted_indexes.cmake [=[
    file(STRINGS ${CMAKE_CURRENT_BINARY_DIR}/${SOURCE}_lookup.c raw_items
        REGEX ",.*ITEM|ITEM_V(\\1)")
    foreach(line IN LISTS raw_items)
        string(REGEX REPLACE "^.*\"([a-z_]+)\".*$" "\\1" item ${line})
        list(APPEND items ${item})
    endforeach()
    list(SORT items)

    file(READ ${CMAKE_CURRENT_BINARY_DIR}/${SOURCE}_lookup.c raw_file)
    list(LENGTH items items_length)
    math(EXPR items_length "${items_length} - 1")
    foreach(index RANGE ${items_length})
        list(GET items ${index} item)
        string(REGEX REPLACE "((ITEM|ITEM_V)\\(${item},)" " ${index}, \\1" raw_file "${raw_file}")
    endforeach()

    file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/${SOURCE}_lookup.c "${raw_file}")
]=])

# Function which adds custom commands to generate c files from gperf files
function(process_gperf out_var)
    set(result)
    foreach(in_f ${ARGN})
        get_filename_component(source ${in_f} NAME_WE)
        set(in_f ${CMAKE_CURRENT_SOURCE_DIR}/${in_f})
        set(out_f "${CMAKE_CURRENT_BINARY_DIR}/${source}_lookup.c")
        set(out_count_f "${CMAKE_CURRENT_BINARY_DIR}/${source}_count.c")
        add_custom_command(OUTPUT ${out_f}
            COMMAND gperf --output-file ${out_f} < ${in_f}
            COMMAND ${CMAKE_COMMAND} -DSOURCE=${source} -P ${CMAKE_CURRENT_BINARY_DIR}/add_sorted_indexes.cmake
            COMMAND ${CMAKE_COMMAND} -DSOURCE=${source} -P ${CMAKE_CURRENT_BINARY_DIR}/total_keywords.cmake
            DEPENDS ${in_f}
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            COMMENT "Creating preprocessed file ${out_f}"
            VERBATIM
        )
        list(APPEND result ${out_f})
    endforeach()
    set(${out_var} "${result}" PARENT_SCOPE)
endfunction()

process_gperf(gperf_processed_files
    src/confitems.gperf
    src/envtoconfitems.gperf
)

set(ccache_sources
    src/args.c
    src/ccache.c
    src/cleanup.c
    src/common_header.c
    src/compopt.c
    src/compress.c
    src/compression.c
    src/compr_none.c
    src/compr_zstd.c
    src/conf.c
    src/confitems.c
    src/counters.c
    src/decompr_none.c
    src/decompr_zstd.c
    src/execute.c
    src/exitfn.c
    src/hash.c
    src/hashutil.c
    src/language.c
    src/lockfile.c
    src/manifest.c
    src/mdfour.c
    src/result.c
    src/stats.c
    src/unify.c
    src/util.c
    src/third_party/getopt_long.c
    src/third_party/hashtable.c
    src/third_party/hashtable_itr.c
    src/third_party/minitrace.c
    src/third_party/xxhash.c
    ${CMAKE_BINARY_DIR}/version.c
)

add_library(ccache_object_lib OBJECT ${ccache_sources} ${gperf_processed_files})
target_include_directories(ccache_object_lib PUBLIC ${CMAKE_BINARY_DIR} src/)
target_link_libraries(ccache_object_lib PUBLIC ZSTD::ZSTD)

add_executable(ccache src/main.c)
target_link_libraries(ccache PRIVATE ccache_object_lib)

#
# unittest
#

enable_testing()
set(unittest_files
    unittest/framework.c
    unittest/main.c
    unittest/test_args.c
    unittest/test_argument_processing.c
    unittest/test_compopt.c
    unittest/test_compr_none.c
    unittest/test_compr_zstd.c
    unittest/test_conf.c
    unittest/test_counters.c
    unittest/test_hash.c
    unittest/test_hashutil.c
    unittest/test_lockfile.c
    unittest/test_stats.c
    unittest/test_util.c
    unittest/util.c
)

file(WRITE ${CMAKE_BINARY_DIR}/suites.h.in)
foreach(source IN LISTS unittest_files)
    file(STRINGS ${source} source_test_suite REGEX "TEST_SUITE\\(")
    string(REGEX MATCH "([a-z]+_[a-z]+_[a-z]+)|([a-z]+_[a-z]+)|([a-z]+)" suite_name "${source_test_suite}")
    if (suite_name)
        file(APPEND ${CMAKE_BINARY_DIR}/suites.h.in "SUITE(${suite_name})\n")
    endif()
endforeach()
configure_file(${CMAKE_BINARY_DIR}/suites.h.in ${CMAKE_BINARY_DIR}/suites.h)

add_executable(unittest ${unittest_files})
target_link_libraries(unittest PRIVATE ccache_object_lib)

add_test(NAME unittest COMMAND unittest)

include(CheckIncludeFile)
foreach(include_file IN ITEMS
        ctype.h
        dirent.h
        inttypes.h
        locale.h
        memory.h
        ndir.h
        pwd.h
        stdarg.h
        stddef.h
        stdint.h
        stdlib.h
        strings.h
        string.h
        sys/dir.h
        sys/mman.h
        sys/ndir.h
        sys/stat.h
        sys/time.h
        sys/types.h
        sys/wait.h
        termios.h
        unistd.h
        varargs.h
    )
    string(TOUPPER ${include_file} include_var)
    string(REGEX REPLACE ".|/" "_" include_var ${include_var})
    set(include_var HAVE_${include_var})
    check_include_file(${include_file} ${include_var})
endforeach()

include(CheckFunctionExists)
foreach(func IN ITEMS
        asprintf
        GetFinalPathNameByHandleW
        getopt_long
        getpwuid
        gettimeofday
        localeconv
        localtime_r
        mkstemp
        realpath
        snprintf
        strndup
        strtok_r
        unsetenv
        utimes
        vasprintf
        va_copy
        vsnprintf
        __va_copy
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
foreach(type IN ITEMS
        intmax_t
        uintmax_t
        uintptr_t
        ptrdiff_t
        "long long"
        "long long int"
        "unsigned long long int"
    )
    string(TOUPPER ${type} type_var)
    string(REGEX REPLACE " " "_" type_var ${type_var})
    set(type_var HAVE_${type_var})
    check_type_size("${type}" ${type_var})
endforeach()

include(CheckStructHasMember)
check_struct_has_member("struct lconv" "decimal_point" locale.h HAVE_STRUCT_LCONV_DECIMAL_POINT)
check_struct_has_member("struct lconv" "thousands_sep" locale.h HAVE_STRUCT_LCONV_THOUSANDS_SEP)

if(WIN32)
    set(_WIN32_WINNT 0x0600)
    target_link_libraries(ccache PRIVATE ws2_32)
    target_link_libraries(unittest PRIVATE ws2_32)
endif()

configure_file(config.h.cmake.in ${CMAKE_BINARY_DIR}/config.h)

if(NOT HAVE_SNPRINTF)
    target_sources(ccache PRIVATE src/third_party/snprintf.c)
endif()
