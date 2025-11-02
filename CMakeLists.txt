cmake_minimum_required(VERSION 3.15)

# CRITICAL: Strip problematic flags BEFORE any compiler detection
# These flags are incompatible with NVPTX GPU offloading
set(CMAKE_CXX_FLAGS_INIT "")
set(CMAKE_C_FLAGS_INIT "")

project(
  ${SKBUILD_PROJECT_NAME}
  VERSION ${SKBUILD_PROJECT_VERSION}
  LANGUAGES CXX C
  )

# Helper function to strip unsupported flags
function(strip_unsupported_flags out_var in_var)
  set(tmp "${${in_var}}")
  # remove -fcf-protection variants and any -flto occurrences
  string(REGEX REPLACE "-fcf-protection[^ ]*" "" tmp "${tmp}")
  string(REGEX REPLACE "-flto[^ ]*" "" tmp "${tmp}")
  string(REGEX REPLACE "-fstack-protector[^ ]*" "" tmp "${tmp}")
  string(REGEX REPLACE "-fstack-clash-protection" "" tmp "${tmp}")
  # trim repeated spaces
  string(REGEX REPLACE "[ 	]+" " " tmp "${tmp}")
  string(STRIP "${tmp}" tmp)
  set(${out_var} "${tmp}" PARENT_SCOPE)
endfunction()

# Clean all flag variables immediately after project()
strip_unsupported_flags(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
strip_unsupported_flags(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")
strip_unsupported_flags(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
strip_unsupported_flags(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
strip_unsupported_flags(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO}")
strip_unsupported_flags(CMAKE_CXX_FLAGS_MINSIZEREL "${CMAKE_CXX_FLAGS_MINSIZEREL}")
strip_unsupported_flags(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS}")
strip_unsupported_flags(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS}")
strip_unsupported_flags(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS}")

set(CMAKE_POSITION_INDEPENDENT_CODE TRUE)

# Windows setup
if(MSVC)
  add_definitions(/MP /MT)
endif()
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# pybind11
find_package(Python REQUIRED COMPONENTS Interpreter Development.Module)
find_package(pybind11 CONFIG REQUIRED)

# OpenMP (find before discamb to ensure consistent configuration)
find_package(OpenMP)

option(USE_AMDGPU "Use AMDGPU backend for OpenMP offload" OFF)

# Configure OpenMP GPU offloading flags if OpenMP is found
if(OpenMP_FOUND)
  add_definitions(-DUSE_GPU_OFFLOAD)

  if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    if(USE_AMDGPU)
      # GCC with amdgcn offloading
      set(_omf "-fopenmp -foffload=amdgcn-amdhsa- -fno-lto")
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${_omf}")
      set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fno-lto")
      set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fno-lto")
      set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} -fno-lto")
    else()
      # GCC with nvptx offloading
      # CRITICAL: Add -fno-lto to disable LTO and avoid pulling CF protection into offload
      set(_omf "-fopenmp -foffload=-lm -fno-lto")
      # Add path to cuda toolkit
      set(_omf "${_omf} -I/usr/local/cuda/include")
      set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -L/usr/local/cuda/lib64")
      set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -L/usr/local/cuda/lib64")
      set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} -L/usr/local/cuda/lib64")
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${_omf}")
      # Also disable LTO in linker flags
      set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fno-lto")
      set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fno-lto")
      set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} -fno-lto")
    endif()
  elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang" OR CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang")
    if(USE_AMDGPU)
      set(_omf "-fopenmp -fopenmp-targets=amdgcn-amd-amdhsa")
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${_omf}")
    else()
      set(_omf "-fopenmp -fopenmp-targets=nvptx64-nvidia-cuda")
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${_omf}")
    endif()
  elseif(CMAKE_CXX_COMPILER_ID STREQUAL "IntelLLVM")
    set(_omf "-fopenmp -fopenmp-targets=spir64")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${_omf}")
  endif()
endif()

# discamb
option(USE_PRECOMPILED_DISCAMB "Whether to look for a compiled DiSCaMB library. Will compile as normal if not found" OFF)

if (USE_PRECOMPILED_DISCAMB)
  FIND_LIBRARY(DISCAMB_LIB discamb PATHS
    ${PROJECT_BINARY_DIR}/build/lib
    ${PROJECT_SOURCE_DIR}/lib/discamb/build/build/lib
    ${PROJECT_SOURCE_DIR}/lib/discamb/build/build/lib/Release
  )
  if(NOT DISCAMB_LIB)
    message("DiSCaMB not found, building...")
    option(DISCAMB_BUILD_EXAMPLES OFF)
    option(BUILD_SHARED_LIBS ON)
    add_subdirectory(lib/discamb)
  else()
    message("DiSCaMB found: ${DISCAMB_LIB}")
    add_library(discamb SHARED IMPORTED)
    set_target_properties(discamb PROPERTIES IMPORTED_LOCATION ${DISCAMB_LIB})
    configure_file ("${PROJECT_SOURCE_DIR}/lib/discamb/config.h.in" "${PROJECT_BINARY_DIR}/lib/discamb/build/include/discamb/config.h")
  endif()
else()
  option(DISCAMB_BUILD_EXAMPLES OFF)
  option(BUILD_SHARED_LIBS ON)
  add_subdirectory(lib/discamb)
endif()

# Fix import issue with pybind11_json
# Discamb has its external dependencies in a "third-party" folder, with some modifications.
# The "nlohmann" folder prefix is not used there.
# This hack should fix it when compiling, but there is still an annoying include error in the IDE
file(COPY "${CMAKE_CURRENT_SOURCE_DIR}/lib/discamb/third-party/json.hpp" DESTINATION lib/nlohmann/include/nlohmann)
include_directories("${CMAKE_CURRENT_BINARY_DIR}/lib/nlohmann/include")

# python module
python_add_library(_cpp_module MODULE
  src/python_module.cpp
  src/PythonInterface.cpp
  src/DiscambStructureFactorCalculator.cpp
  src/scattering_table.cpp
  src/atom_assignment.cpp
  src/read_structure.cpp
  src/TimedInterface.cpp
  src/tests.cpp
)

target_link_libraries(_cpp_module PRIVATE pybind11::headers)
if(OpenMP_FOUND)
  target_link_libraries(_cpp_module PRIVATE OpenMP::OpenMP_CXX)
endif()
target_link_libraries(_cpp_module PUBLIC discamb)
target_include_directories(_cpp_module PRIVATE lib/pybind11_json/include)
# For some reason, the include directories are not propagated when linking
get_target_property(DISCAMB_INCLUDE_DIRECTORIES discamb INCLUDE_DIRECTORIES)
target_include_directories(_cpp_module PUBLIC ${DISCAMB_INCLUDE_DIRECTORIES})
target_include_directories(_cpp_module PUBLIC include)
# pybind11 required flag
set_target_properties(_cpp_module PROPERTIES CXX_VISIBILITY_PRESET hidden)

install(TARGETS _cpp_module DESTINATION pydiscamb)

# Install stub
file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/pydiscamb/_cpp_module.pyi DESTINATION pydiscamb)

# TAAM databank
file(GLOB_RECURSE TAAM_DATA_FILES "${PROJECT_SOURCE_DIR}/data/*databank.txt")
foreach(TAAM_BANK ${TAAM_DATA_FILES})
  install(FILES ${TAAM_BANK} DESTINATION pydiscamb/data)
endforeach(TAAM_BANK ${TAAM_DATA_FILES})