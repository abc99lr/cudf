#=============================================================================
# Copyright (c) 2018-2019, NVIDIA CORPORATION.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#=============================================================================
cmake_minimum_required(VERSION 3.12 FATAL_ERROR)

project(CUDA_DATAFRAME VERSION 0.11.0 LANGUAGES C CXX CUDA)

if(NOT CMAKE_CUDA_COMPILER)
  message(SEND_ERROR "CMake cannot locate a CUDA compiler")
endif(NOT CMAKE_CUDA_COMPILER)

###################################################################################################
# - build type ------------------------------------------------------------------------------------

# Set a default build type if none was specified
set(DEFAULT_BUILD_TYPE "Release")

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to '${DEFAULT_BUILD_TYPE}' since none specified.")
  set(CMAKE_BUILD_TYPE "${DEFAULT_BUILD_TYPE}" CACHE
      STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
    "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)

###################################################################################################
# - compiler options ------------------------------------------------------------------------------

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_C_COMPILER $ENV{CC})
set(CMAKE_CXX_COMPILER $ENV{CXX})
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_CUDA_STANDARD 14)
set(CMAKE_CUDA_STANDARD_REQUIRED ON)

if(CMAKE_COMPILER_IS_GNUCXX)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror")

    option(CMAKE_CXX11_ABI "Enable the GLIBCXX11 ABI" ON)
    if(CMAKE_CXX11_ABI)
        message(STATUS "CUDF: Enabling the GLIBCXX11 ABI")
    else()
        message(STATUS "CUDF: Disabling the GLIBCXX11 ABI")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_GLIBCXX_USE_CXX11_ABI=0")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_GLIBCXX_USE_CXX11_ABI=0")
        set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -Xcompiler -D_GLIBCXX_USE_CXX11_ABI=0")
    endif(CMAKE_CXX11_ABI)
endif(CMAKE_COMPILER_IS_GNUCXX)

if(CMAKE_CUDA_COMPILER_VERSION)
  # Compute the version. from  CMAKE_CUDA_COMPILER_VERSION
  string(REGEX REPLACE "([0-9]+)\\.([0-9]+).*" "\\1" CUDA_VERSION_MAJOR ${CMAKE_CUDA_COMPILER_VERSION})
  string(REGEX REPLACE "([0-9]+)\\.([0-9]+).*" "\\2" CUDA_VERSION_MINOR ${CMAKE_CUDA_COMPILER_VERSION})
  set(CUDA_VERSION "${CUDA_VERSION_MAJOR}.${CUDA_VERSION_MINOR}" CACHE STRING "Version of CUDA as computed from nvcc.")
  mark_as_advanced(CUDA_VERSION)
endif()

message(STATUS "CUDA_VERSION_MAJOR: ${CUDA_VERSION_MAJOR}")
message(STATUS "CUDA_VERSION_MINOR: ${CUDA_VERSION_MINOR}")
message(STATUS "CUDA_VERSION: ${CUDA_VERSION}")

# Always set this convenience variable
set(CUDA_VERSION_STRING "${CUDA_VERSION}")

# Auto-detect available GPU compute architectures
set(GPU_ARCHS "ALL" CACHE STRING
  "List of GPU architectures (semicolon-separated) to be compiled for. Pass 'ALL' if you want to compile for all supported GPU architectures. Empty string means to auto-detect the GPUs on the current system")

if("${GPU_ARCHS}" STREQUAL "")
  include(cmake/EvalGpuArchs.cmake)
  evaluate_gpu_archs(GPU_ARCHS)
endif()

if("${GPU_ARCHS}" STREQUAL "ALL")
  set(GPU_ARCHS "60")
  if((CUDA_VERSION_MAJOR EQUAL 9) OR (CUDA_VERSION_MAJOR GREATER 9))
    set(GPU_ARCHS "${GPU_ARCHS};70")
  endif()
  if((CUDA_VERSION_MAJOR EQUAL 10) OR (CUDA_VERSION_MAJOR GREATER 10))
    set(GPU_ARCHS "${GPU_ARCHS};75")
  endif()
endif()
message("GPU_ARCHS = ${GPU_ARCHS}")

foreach(arch ${GPU_ARCHS})
  set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -gencode arch=compute_${arch},code=sm_${arch}")
endforeach()

list(GET GPU_ARCHS -1 ptx)
set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -gencode arch=compute_${ptx},code=compute_${ptx}")

set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} --expt-extended-lambda --expt-relaxed-constexpr")

# set warnings as errors
# TODO: remove `no-maybe-unitialized` used to suppress warnings in rmm::exec_policy
set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -Werror cross-execution-space-call -Xcompiler -Wall,-Werror")

# Option to enable line info in CUDA device compilation to allow introspection when profiling / memchecking
option(CMAKE_CUDA_LINEINFO "Enable the -lineinfo option for nvcc (useful for cuda-memcheck / profiler" OFF)
if(CMAKE_CUDA_LINEINFO)
    set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -lineinfo")
endif(CMAKE_CUDA_LINEINFO)

# Debug options
if(CMAKE_BUILD_TYPE MATCHES Debug)
    message(STATUS "Building with debugging flags")
    set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -G -Xcompiler -rdynamic")
endif(CMAKE_BUILD_TYPE MATCHES Debug)

# To apply RUNPATH to transitive dependencies (this is a temporary solution)
set(CMAKE_SHARED_LINKER_FLAGS "-Wl,--disable-new-dtags")
set(CMAKE_EXE_LINKER_FLAGS "-Wl,--disable-new-dtags")

# Build options
option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(BUILD_TESTS "Configure CMake to build tests" ON)
option(BUILD_BENCHMARKS "Configure CMake to build (google) benchmarks" OFF)

###################################################################################################
# - cmake modules ---------------------------------------------------------------------------------

set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules/" ${CMAKE_MODULE_PATH})

include(FeatureSummary)
include(CheckIncludeFiles)
include(CheckLibraryExists)

###################################################################################################
# - conda environment -----------------------------------------------------------------------------

if("$ENV{CONDA_BUILD}" STREQUAL "1")
    set(CMAKE_SYSTEM_PREFIX_PATH "$ENV{BUILD_PREFIX};$ENV{PREFIX};${CMAKE_SYSTEM_PREFIX_PATH}")
    set(CONDA_INCLUDE_DIRS "$ENV{BUILD_PREFIX}/include" "$ENV{PREFIX}/include")
    set(CONDA_LINK_DIRS "$ENV{BUILD_PREFIX}/lib" "$ENV{PREFIX}/lib")
    message(STATUS "Conda build detected, CMAKE_SYSTEM_PREFIX_PATH set to: ${CMAKE_SYSTEM_PREFIX_PATH}")
elseif(DEFINED ENV{CONDA_PREFIX})
    set(CMAKE_SYSTEM_PREFIX_PATH "$ENV{CONDA_PREFIX};${CMAKE_SYSTEM_PREFIX_PATH}")
    set(CONDA_INCLUDE_DIRS "$ENV{CONDA_PREFIX}/include")
    set(CONDA_LINK_DIRS "$ENV{CONDA_PREFIX}/lib")
    message(STATUS "Conda environment detected, CMAKE_SYSTEM_PREFIX_PATH set to: ${CMAKE_SYSTEM_PREFIX_PATH}")
endif("$ENV{CONDA_BUILD}" STREQUAL "1")

###################################################################################################
# - find arrow ------------------------------------------------------------------------------------

message(STATUS "BUILDING ARROW")
include(ConfigureArrow)

if(ARROW_FOUND)
    message(STATUS "Apache Arrow found in ${ARROW_INCLUDE_DIR}")
else()
    message(FATAL_ERROR "Apache Arrow not found, please check your settings.")
endif(ARROW_FOUND)

###################################################################################################
# - find zlib -------------------------------------------------------------------------------------

find_package(ZLIB REQUIRED)

message(STATUS "ZLIB: ZLIB_LIBRARIES set to ${ZLIB_LIBRARIES}")
message(STATUS "ZLIB: ZLIB_INCLUDE_DIRS set to ${ZLIB_INCLUDE_DIRS}")

if(ZLIB_FOUND)
    message(STATUS "ZLib found in ${ZLIB_INCLUDE_DIRS}")
else()
    message(FATAL_ERROR "ZLib not found, please check your settings.")
endif(ZLIB_FOUND)

###################################################################################################
# - find boost ------------------------------------------------------------------------------------

# Don't look for a CMake configuration file
set(Boost_NO_BOOST_CMAKE ON)

find_package(
    Boost REQUIRED MODULE
    COMPONENTS filesystem
)

message(STATUS "BOOST: Boost_LIBRARIES set to ${Boost_LIBRARIES}")
message(STATUS "BOOST: Boost_INCLUDE_DIRS set to ${Boost_INCLUDE_DIRS}")

if(Boost_FOUND)
    message(STATUS "Boost found in ${Boost_INCLUDE_DIRS}")
else()
    message(FATAL_ERROR "Boost not found, please check your settings.")
endif(Boost_FOUND)

###################################################################################################
# - RMM -------------------------------------------------------------------------------------------

find_path(RMM_INCLUDE "rmm"
          HINTS "$ENV{RMM_ROOT}/include")

find_library(RMM_LIBRARY "rmm"
             HINTS "$ENV{RMM_ROOT}/lib" "$ENV{RMM_ROOT}/build")

message(STATUS "RMM: RMM_LIBRARY set to ${RMM_LIBRARY}")
message(STATUS "RMM: RMM_INCLUDE set to ${RMM_INCLUDE}")

add_library(rmm SHARED IMPORTED ${RMM_LIBRARY})
if(RMM_INCLUDE AND RMM_LIBRARY)
    set_target_properties(rmm PROPERTIES IMPORTED_LOCATION ${RMM_LIBRARY})
endif(RMM_INCLUDE AND RMM_LIBRARY)

###################################################################################################
# - DLPACK -------------------------------------------------------------------------------------------

find_path(
    DLPACK_INCLUDE "dlpack"
    HINTS "$ENV{DLPACK_ROOT}/include"
)

message(STATUS "DLPACK: DLPACK_INCLUDE set to ${DLPACK_INCLUDE}")

###################################################################################################
# - jitify ----------------------------------------------------------------------------------------

option(JITIFY_USE_CACHE "Use a file cache for JIT compiled kernels" ON)
if(JITIFY_USE_CACHE)
    message(STATUS "Using file cache for JIT compiled kernels")
    add_definitions("-DJITIFY_USE_CACHE -DCUDF_VERSION=${CMAKE_PROJECT_VERSION}")
endif(JITIFY_USE_CACHE)

###################################################################################################
# - add gtest -------------------------------------------------------------------------------------

if(BUILD_TESTS)
    include(CTest)
    include(ConfigureGoogleTest)

    if(GTEST_FOUND)
        message(STATUS "Google C++ Testing Framework (Google Test) found in ${GTEST_ROOT}")
        include_directories(${GTEST_INCLUDE_DIR})
        add_subdirectory(${CMAKE_SOURCE_DIR}/tests)
        add_subdirectory(${CMAKE_SOURCE_DIR}/custrings/tests)
    else()
        message(AUTHOR_WARNING "Google C++ Testing Framework (Google Test) not found: automated tests are disabled.")
    endif(GTEST_FOUND)
endif(BUILD_TESTS)

message(STATUS "CUDF_TEST_LIST set to: ${CUDF_TEST_LIST}")
message(STATUS "NVSTRINGS_TEST_LIST set to: ${NVSTRINGS_TEST_LIST}")

###################################################################################################
# - add google benchmark --------------------------------------------------------------------------

if(BUILD_BENCHMARKS)

  include(ConfigureGoogleBenchmark)

  if(GBENCH_FOUND)
    message(STATUS "Google C++ Benchmarking Framework (Google Benchmark) found in ${GBENCH_ROOT}")
    include_directories(${GBENCH_INCLUDE_DIR})
    add_subdirectory(${CMAKE_SOURCE_DIR}/benchmarks)
  else()
    message(AUTHOR_WARNING "Google C++ Benchmarking Framework (Google Benchmark) not found: automated tests are disabled.")
  endif(GBENCH_FOUND)

endif(BUILD_BENCHMARKS)

###################################################################################################
# - include paths ---------------------------------------------------------------------------------

if(CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES)
	include_directories("${CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES}")
endif(CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES)

include_directories("${CMAKE_BINARY_DIR}/include"
                    "${CMAKE_SOURCE_DIR}/include"
                    "${CMAKE_SOURCE_DIR}/src"
                    "${CMAKE_SOURCE_DIR}/thirdparty/cub"
                    "${CMAKE_SOURCE_DIR}/thirdparty/jitify"
                    "${CMAKE_SOURCE_DIR}/thirdparty/libcudacxx/include"
                    "${ARROW_INCLUDE_DIR}"
                    "${FLATBUFFERS_INCLUDE_DIR}"
                    "${ZLIB_INCLUDE_DIRS}"
                    "${Boost_INCLUDE_DIRS}"
                    "${RMM_INCLUDE}"
                    "${DLPACK_INCLUDE}")

if(CONDA_INCLUDE_DIRS)
    include_directories("${CONDA_INCLUDE_DIRS}")
endif(CONDA_INCLUDE_DIRS)

###################################################################################################
# - library paths ---------------------------------------------------------------------------------

link_directories("${CMAKE_CUDA_IMPLICIT_LINK_DIRECTORIES}" # CMAKE_CUDA_IMPLICIT_LINK_DIRECTORIES is an undocumented/unsupported variable containing the link directories for nvcc
                 "${CMAKE_BINARY_DIR}/lib"
                 "${CMAKE_BINARY_DIR}"
                 "${FLATBUFFERS_LIBRARY_DIR}"
                 "${GTEST_LIBRARY_DIR}"
                 "${RMM_LIBRARY}")

if(CONDA_LINK_DIRS)
    link_directories("${CONDA_LINK_DIRS}")
endif(CONDA_LINK_DIRS)

###################################################################################################
# - library targets -------------------------------------------------------------------------------

add_library(libNVStrings
            custrings/strings/NVStrings.cu
            custrings/strings/NVStringsImpl.cu
            custrings/strings/array.cu
            custrings/strings/attrs.cu
            custrings/strings/case.cu
            custrings/strings/combine.cu
            custrings/strings/convert.cu
            custrings/strings/count.cu
            custrings/strings/datetime.cu
            custrings/strings/extract.cu
            custrings/strings/extract_record.cu
            custrings/strings/find.cu
            custrings/strings/findall.cu
            custrings/strings/findall_record.cu
            custrings/strings/modify.cu
            custrings/strings/pad.cu
            custrings/strings/replace.cu
            custrings/strings/replace_backref.cu
            custrings/strings/replace_multi.cu
            custrings/strings/split.cu
            custrings/strings/strip.cu
            custrings/strings/substr.cu
            custrings/strings/urlencode.cu
            custrings/util.cu
            custrings/regex/regexec.cpp
            custrings/regex/regcomp.cpp)

add_library(libNVCategory
            custrings/category/NVCategory.cu
            custrings/category/numeric_category.cu
            custrings/category/numeric_category_int.cu
            custrings/category/numeric_category_long.cu
            custrings/category/numeric_category_float.cu
            custrings/category/numeric_category_double.cu)

add_library(libNVText
            custrings/text/NVText.cu
            custrings/text/edit_distance.cu
            custrings/text/ngram.cu
            custrings/text/stemmer.cu
            custrings/text/tokens.cu
            custrings/util.cu)

add_library(cudf
            src/comms/ipc/ipc.cpp
            src/column/legacy/column.cpp
            src/column/legacy/context.cpp
            src/table/legacy/table.cpp
            src/strings/nvcategory_util.cpp
            src/join/legacy/joining.cu
            src/orderby/legacy/orderby.cu
            src/predicates/is_sorted.cu
            src/sort/legacy/digitize.cu
            src/groupby/hash/legacy/groupby.cu
            src/groupby/sort/legacy/sort_helper.cu
            src/groupby/sort/legacy/groupby.cu
            src/groupby/legacy/groupby_without_aggregation.cu
            src/groupby/common/legacy/aggregation_requests.cpp
            src/rolling/rolling.cu
            src/rolling/jit/code/kernel.cpp
            src/rolling/jit/code/operation.cpp
            src/rolling/jit/util/type.cpp
            src/binaryop/legacy/binaryop.cpp
            src/binaryop/legacy/compiled/binary_ops.cu
            src/binaryop/legacy/jit/code/kernel.cpp
            src/binaryop/legacy/jit/code/operation.cpp
            src/binaryop/legacy/jit/code/traits.cpp
            src/binaryop/legacy/jit/util/operator.cpp
            src/binaryop/legacy/jit/util/type.cpp
            src/jit/legacy/type.cpp
            src/jit/parser.cpp
            src/jit/cache.cpp
            src/jit/launcher.cpp
            src/transform/legacy/transform.cpp
            src/transform/jit/code/kernel.cpp
            src/transform/legacy/nans_to_nulls.cu
            src/bitmask/legacy/bitmask_ops.cu
            src/stream_compaction/legacy/apply_boolean_mask.cu
            src/stream_compaction/legacy/drop_nulls.cu
            src/stream_compaction/legacy/drop_duplicates.cu
            src/datetime/legacy/datetime_ops.cu
            src/datetime/datetime_util.cpp
            src/hash/legacy/hashing.cu
            src/quantiles/quantiles.cu
            src/quantiles/group_quantiles.cu
            src/reductions/legacy/reductions.cu
            src/reductions/legacy/min.cu
            src/reductions/legacy/max.cu
            src/reductions/legacy/any.cu
            src/reductions/legacy/all.cu
            src/reductions/legacy/sum.cu
            src/reductions/legacy/product.cu
            src/reductions/legacy/sum_of_squares.cu
            src/reductions/legacy/mean.cu
            src/reductions/legacy/var.cu
            src/reductions/legacy/std.cu
            src/reductions/legacy/scan.cu
            src/replace/legacy/replace.cu
            src/reshape/stack.cu
            src/transpose/legacy/transpose.cu
            src/merge/legacy/merge.cu
            src/unary/legacy/math_ops.cu
            src/unary/legacy/cast_ops.cu
            src/unary/legacy/null_ops.cu
            src/io/legacy/cuio_common.cpp
            src/io/legacy/io_functions.cpp
            src/io/convert/csr/legacy/cudf_to_csr.cu
            src/io/convert/dlpack/legacy/cudf_dlpack.cpp
            src/io/avro/legacy/avro_reader_impl.cu
            src/io/avro/avro_gpu.cu
            src/io/avro/avro.cpp
            src/io/csv/legacy/csv_reader_impl.cu
            src/io/csv/legacy/csv_writer.cu
            src/io/csv/legacy/csv_gpu.cu
            src/io/json/legacy/json_reader_impl.cu
            src/io/orc/legacy/orc_reader_impl.cu
            src/io/orc/legacy/orc_writer_impl.cu
            src/io/orc/orc.cpp
            src/io/orc/timezone.cpp
            src/io/orc/stripe_data.cu
            src/io/orc/stripe_init.cu
            src/io/orc/stripe_enc.cu
            src/io/orc/dict_enc.cu
            src/io/parquet/page_data.cu
            src/io/parquet/page_hdr.cu
            src/io/parquet/legacy/parquet_reader_impl.cu
            src/io/parquet/parquet.cpp
            src/io/comp/cpu_unbz2.cpp
            src/io/comp/uncomp.cpp
            src/io/comp/brotli_dict.cpp
            src/io/comp/debrotli.cu
            src/io/comp/snap.cu
            src/io/comp/unsnap.cu
            src/io/comp/gpuinflate.cu
            src/io/utilities/datasource.cpp
            src/io/utilities/legacy/parsing_utils.cu
            src/utilities/legacy/cuda_utils.cu
            src/utilities/column_utils.cpp
            src/utilities/legacy/error_utils.cpp
            src/utilities/nvtx/nvtx_utils.cpp
            src/utilities/nvtx/legacy/nvtx_utils.cpp
            src/copying/copy.cpp
            src/copying/slice.cpp
            src/copying/split.cpp
            src/copying/legacy/copy.cpp
            src/copying/legacy/gather.cu
            src/copying/legacy/scatter.cu
            src/copying/legacy/slice.cu
            src/copying/legacy/split.cu
            src/bitmask/legacy/legacy_bitmask.cpp
            src/copying/legacy/copy_range.cu
            src/filling/legacy/fill.cu
            src/filling/legacy/repeat.cu
            src/filling/legacy/tile.cu
            src/search/legacy/search.cu
            src/column/column.cu
            src/column/column_view.cpp
            src/column/column_device_view.cu
            src/column/column_factories.cpp
            src/table/table_view.cpp
            src/table/table_device_view.cu
            src/table/table.cpp
            src/bitmask/null_mask.cu
            src/sort/sort.cu
            src/strings/strings_column_factories.cu
            src/strings/strings_column_view.cu
            src/strings/utilities.cu
            src/strings/copying/copying.cu
            src/strings/sorting/sorting.cu
            src/column/legacy/interop.cpp)

# Rename installation to proper names for later finding
set_target_properties(libNVStrings PROPERTIES OUTPUT_NAME "NVStrings")
set_target_properties(libNVCategory PROPERTIES OUTPUT_NAME "NVCategory")
set_target_properties(libNVText PROPERTIES OUTPUT_NAME "NVText")

# Override RPATH for cudf
set_target_properties(cudf PROPERTIES BUILD_RPATH "\$ORIGIN")

# Override RPATH for nvstrings
set_target_properties(libNVStrings PROPERTIES BUILD_RPATH "\$ORIGIN")
set_target_properties(libNVCategory PROPERTIES BUILD_RPATH "\$ORIGIN")
set_target_properties(libNVText PROPERTIES BUILD_RPATH "\$ORIGIN")

###################################################################################################
# - jitify ----------------------------------------------------------------------------------------

# Creates executable stringify and uses it to convert types.h to c-str for use in JIT code
add_executable(stringify "${CMAKE_SOURCE_DIR}/thirdparty/jitify/stringify.cpp")
execute_process(WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/include)

add_custom_command(OUTPUT ${CMAKE_BINARY_DIR}/include/types.h.jit
                   WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include
                   COMMAND ${CMAKE_BINARY_DIR}/stringify cudf/types.h > ${CMAKE_BINARY_DIR}/include/types.h.jit
                   COMMENT "Run stringify on header types.h to convert it to c-str for use in JIT compiled code"
                   DEPENDS stringify
                   MAIN_DEPENDENCY ${CMAKE_CURRENT_SOURCE_DIR}/include/cudf/types.h)

add_custom_command(OUTPUT ${CMAKE_BINARY_DIR}/include/types.hpp.jit
                   WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include
                   COMMAND ${CMAKE_BINARY_DIR}/stringify cudf/types.hpp > ${CMAKE_BINARY_DIR}/include/types.hpp.jit
                   COMMENT "Run stringify on header types.hpp to convert it to c-str for use in JIT compiled code"
                   DEPENDS stringify
                   MAIN_DEPENDENCY ${CMAKE_CURRENT_SOURCE_DIR}/include/cudf/types.hpp)

add_custom_target(stringify_run DEPENDS
                  ${CMAKE_BINARY_DIR}/include/types.h.jit
                  ${CMAKE_BINARY_DIR}/include/types.hpp.jit)

add_dependencies(cudf stringify_run)

###################################################################################################
# - build options ---------------------------------------------------------------------------------

option(USE_NVTX "Build with NVTX support" ON)
if(USE_NVTX)
    message(STATUS "Using Nvidia Tools Extension")
    find_library(NVTX_LIBRARY nvToolsExt PATH ${CMAKE_CUDA_IMPLICIT_LINK_DIRECTORIES})
    target_link_libraries(cudf ${NVTX_LIBRARY})
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DUSE_NVTX")
endif(USE_NVTX)

option(HT_LEGACY_ALLOCATOR "Use the legacy allocator for hash tables" ON)
if(HT_LEGACY_ALLOCATOR)
    message(STATUS "Using legacy allocator for hash tables")
    set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} --define-macro HT_LEGACY_ALLOCATOR")
endif(HT_LEGACY_ALLOCATOR)

###################################################################################################
# - link libraries --------------------------------------------------------------------------------

# Get all the symbols from the Arrow CUDA Library for Cython
set(ARROW_CUDA_LIB_LINK -Wl,--whole-archive ${ARROW_CUDA_LIB} -Wl,--no-whole-archive)

# link targets for NVStrings
target_link_libraries(libNVStrings rmm cudart cuda)
target_link_libraries(libNVCategory libNVStrings rmm cudart cuda)
target_link_libraries(libNVText libNVStrings rmm cudart cuda)

# link targets for cuDF
target_link_libraries(cudf NVCategory NVStrings rmm ${ARROW_CUDA_LIB_LINK} ${ARROW_LIB} nvrtc cudart cuda ${ZLIB_LIBRARIES} ${Boost_LIBRARIES})

###################################################################################################
# - install targets -------------------------------------------------------------------------------

# install targets for NVStrings
install(TARGETS libNVStrings
        DESTINATION lib
        COMPONENT nvstrings)

install(TARGETS libNVCategory
        DESTINATION lib
        COMPONENT nvstrings)

install(TARGETS libNVText
        DESTINATION lib
        COMPONENT nvstrings)

install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/nvstrings
        DESTINATION include
        COMPONENT nvstrings)

add_custom_target(nvstrings
                  DEPENDS libNVStrings libNVCategory libNVText)

# install targets for cuDF
install(TARGETS cudf
        DESTINATION lib
        COMPONENT cudf)
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/cudf
        DESTINATION include
        COMPONENT cudf)

add_custom_target(install_cudf
                  COMMAND "${CMAKE_COMMAND}" -DCOMPONENT=cudf -P "${CMAKE_BINARY_DIR}/cmake_install.cmake"
                  DEPENDS cudf)

if(BUILD_TESTS)
    add_dependencies(install_cudf cudftestutil)
endif(BUILD_TESTS)

add_custom_target(install_nvstrings
                  COMMAND "${CMAKE_COMMAND}" -DCOMPONENT=nvstrings -P "${CMAKE_BINARY_DIR}/cmake_install.cmake"
                  DEPENDS nvstrings)

add_custom_target(build_tests_cudf
                  DEPENDS ${CUDF_TEST_LIST})

add_custom_target(build_tests_nvstrings
                  DEPENDS ${NVSTRINGS_TEST_LIST})

add_custom_target(test_cudf
                  COMMAND ctest -E "NVSTRINGS"
                  DEPENDS build_tests_cudf)

add_custom_target(test_nvstrings
                  COMMAND ctest -R "NVSTRINGS"
                  DEPENDS build_tests_nvstrings)

###################################################################################################
# - make documentation ----------------------------------------------------------------------------

# doc targets for nvstrings
add_custom_command(OUTPUT NVSTRINGS_DOXYGEN
                   WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/custrings/doxygen
                   COMMAND doxygen Doxyfile
                   VERBATIM
)
add_custom_target(docs_nvstrings DEPENDS NVSTRINGS_DOXYGEN)

# doc targets for cuDF
add_custom_command(OUTPUT CUDF_DOXYGEN
                   WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/doxygen
                   COMMAND doxygen Doxyfile
                   VERBATIM)

add_custom_target(docs_cudf DEPENDS CUDF_DOXYGEN)
