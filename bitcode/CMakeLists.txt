#=============================================================================
#   CMake build system files
#
#   Copyright (c) 2021-23 chipStar developers
#
#   Permission is hereby granted, free of charge, to any person obtaining a copy
#   of this software and associated documentation files (the "Software"), to deal
#   in the Software without restriction, including without limitation the rights
#   to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
#   copies of the Software, and to permit persons to whom the Software is
#   furnished to do so, subject to the following conditions:
#
#   The above copyright notice and this permission notice shall be included in
#   all copies or substantial portions of the Software.
#
#   THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
#   IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
#   FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
#   AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
#   LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
#   OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
#   THE SOFTWARE.
#
#=============================================================================

set(EXTRA_FLAGS)
# Ugly fix for interactions between clang13+ and igc
list(APPEND EXTRA_FLAGS "-cl-no-stdinc")

# disable opaque pointers for LLVM 15 only
if(NOT CLANG_VERSION_LESS_15 AND CLANG_VERSION_LESS_16)
  list(APPEND EXTRA_FLAGS ${DISABLE_OPAQUE_PTRS_OPT})
endif()

set(BC_DESTINATION lib/hip-device-lib)

set(BITCODE_CL_COMPILE_FLAGS
  -Xclang -finclude-default-header -O2 -x cl -cl-std=CL2.0
  -cl-ext=+cl_khr_subgroup
  -DDEFAULT_WARP_SIZE=${DEFAULT_WARP_SIZE} -emit-llvm
  ${EXTRA_FLAGS})
set(BITCODE_C_COMPILE_FLAGS
  -O2 -x c -DDEFAULT_WARP_SIZE=${DEFAULT_WARP_SIZE}
  -emit-llvm ${EXTRA_FLAGS})

# non-OCML sources compiled per-width from chipStar's own bitcode/.
set(NON_OCML_SOURCES "devicelib" "_cl_print_str" "texture" "malloc") # "printf_support"

# Compiles SOURCE treated as OpenCL to LLVM bitcode at the given TARGET triple.
function(add_opencl_bitcode TARGET SOURCE OUTPUT)
  add_custom_command(
    OUTPUT "${OUTPUT}"
    DEPENDS "${SOURCE}"
    COMMAND "${CMAKE_CXX_COMPILER}" ${BITCODE_CL_COMPILE_FLAGS}
    --target=${TARGET} -o "${OUTPUT}" -c "${SOURCE}"
    COMMENT "Building ${SOURCE} for ${TARGET}"
    VERBATIM)
endfunction()

# =============================================================================
# Per-width devicelib build
#
# add_hipspv_devicelib(WIDTH TRIPLE) builds one hipspv-<TRIPLE>.bc by:
#   1. Compiling NON_OCML_SOURCES and c_to_opencl.c at <TRIPLE>
#   2. add_subdirectory()ing ROCm-Device-Libs into a width-scoped binary
#      directory with AMD_DEVICE_LIBS_TARGET_SUFFIX="_w${WIDTH}" so OCML
#      and OCLC targets get a per-width suffix (avoiding name collisions
#      across widths)
#   3. llvm-linking the chipStar bitcodes with OCML/OCLC into the final
#      hip-device-lib/hipspv-<TRIPLE>.bc and installing it
#
# Multiple add_hipspv_devicelib() invocations may coexist in one
# configuration so the same chipStar build can serve both rv32 and rv64
# Vortex devices.  See docs/proposals/chipstar_opencl_32bit_proposal.md
# for the design rationale.
function(add_hipspv_devicelib WIDTH TRIPLE)
  set(_w_dir "${CMAKE_CURRENT_BINARY_DIR}/w${WIDTH}")
  set(_bc_dir "${_w_dir}/BC")
  file(MAKE_DIRECTORY "${_bc_dir}")

  set(_depends "")

  # 1. NON_OCML chipStar OpenCL sources.
  foreach(SOURCE IN LISTS NON_OCML_SOURCES)
    add_opencl_bitcode(
      ${TRIPLE}
      "${CMAKE_CURRENT_SOURCE_DIR}/${SOURCE}.cl"
      "${_bc_dir}/${SOURCE}.bc")
    list(APPEND _depends "${_bc_dir}/${SOURCE}.bc")
  endforeach()

  # 2. c_to_opencl.c -- compiled as C with the SPIR-V target triple.
  add_custom_command(
    OUTPUT "${_bc_dir}/c_to_opencl.bc"
    DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/c_to_opencl.c"
    COMMAND "${CMAKE_CXX_COMPILER}" ${BITCODE_C_COMPILE_FLAGS}
            --target=${TRIPLE}
            -o "${_bc_dir}/c_to_opencl.bc"
            -c "${CMAKE_CURRENT_SOURCE_DIR}/c_to_opencl.c"
    COMMENT "Building c_to_opencl.bc for ${TRIPLE}"
    VERBATIM)
  list(APPEND _depends "${_bc_dir}/c_to_opencl.bc")

  # 3. ROCm-Device-Libs scoped to this width.
  set(SAVED_WARN_DEPRECATED ${CMAKE_WARN_DEPRECATED})
  set(CMAKE_WARN_DEPRECATED OFF)

  # ROCm-Device-Libs reads AMDGPU_TARGET_TRIPLE; with the chipstar branch
  # patches, opencl_bc_lib() and irif/CMakeLists.txt also read
  # AMD_DEVICE_LIBS_TARGET_SUFFIX to suffix CMake target names so multiple
  # invocations don't collide.
  set(AMDGPU_TARGET_TRIPLE ${TRIPLE})
  set(AMD_DEVICE_LIBS_TARGET_SUFFIX "_w${WIDTH}")
  set(ROCM_DEVICELIB_STANDALONE_BUILD ON)
  add_subdirectory(ROCm-Device-Libs
                   "${_w_dir}/ROCm-Device-Libs"
                   EXCLUDE_FROM_ALL)
  set(CMAKE_WARN_DEPRECATED ${SAVED_WARN_DEPRECATED})

  # OCML/OCLC outputs land at ${PROJECT_BINARY_DIR}/amdgcn/bitcode/, which
  # inside the subdir scope resolves to ${_w_dir}/ROCm-Device-Libs/amdgcn/
  # bitcode/. Reference them explicitly here.
  set(_ocml_dir "${_w_dir}/ROCm-Device-Libs/amdgcn/bitcode")
  set(_ocml_lib_targets
    ocml_w${WIDTH}
    oclc_finite_only_off_w${WIDTH}
    oclc_unsafe_math_off_w${WIDTH}
    oclc_correctly_rounded_sqrt_off_w${WIDTH}
    oclc_daz_opt_off_w${WIDTH}
    oclc_isa_version_803_w${WIDTH}
  )
  set(_ocml_lib_bc_paths
    ${_ocml_dir}/ocml.bc
    ${_ocml_dir}/oclc_finite_only_off.bc
    ${_ocml_dir}/oclc_unsafe_math_off.bc
    ${_ocml_dir}/oclc_correctly_rounded_sqrt_off.bc
    ${_ocml_dir}/oclc_daz_opt_off.bc
    ${_ocml_dir}/oclc_isa_version_803.bc
  )

  # 4. Final llvm-link.
  set(_bc_file "hipspv-${TRIPLE}.bc")
  set(_bc_output "${CMAKE_BINARY_DIR}/${BC_DESTINATION}/${_bc_file}")
  add_custom_command(
    OUTPUT "${_bc_output}"
    DEPENDS ${_depends} ${_ocml_lib_targets}
    COMMAND ${CMAKE_COMMAND} -E make_directory
            "${CMAKE_BINARY_DIR}/${BC_DESTINATION}"
    COMMAND "${LLVM_LINK}"
            -o "${_bc_output}"
            ${_depends}
            ${_ocml_lib_bc_paths}
    COMMENT "Linking device library bitcode '${_bc_file}'"
    VERBATIM)

  add_custom_target("devicelib_bc_w${WIDTH}" DEPENDS "${_bc_output}")
  install(FILES "${_bc_output}" DESTINATION ${BC_DESTINATION})
endfunction()

set(_devicelib_per_width_targets "")
foreach(_width _triple IN ZIP_LISTS CHIP_TARGET_POINTER_WIDTHS OFFLOAD_TRIPLES)
  add_hipspv_devicelib(${_width} ${_triple})
  list(APPEND _devicelib_per_width_targets "devicelib_bc_w${_width}")
endforeach()
unset(_width)
unset(_triple)

# Umbrella target consumed by chipStar's top-level CMakeLists.txt.
add_custom_target("devicelib_bc" DEPENDS ${_devicelib_per_width_targets})

# =============================================================================
# Runtime device library (rtdevlib)
#
# A collection of SPIR-V modules which are linked into user's device
# programs based on target capabilities at runtime during JIT compilation.
#
# For example, On OpenCL HIP/CUDA's atomicAdd(float*, float) may be
# implemented with corresponding OpenCL atomic operation if the target
# supports cl_ext_float_atomics extension. In that case the runtime
# would link in a module which implements the atomicAdd(float*, float)
# with the extension's atomic operations. Otherwise, the runtime links
# in a slower, emulated version.
#
# RTDEVLIB_SOURCES* defines OpenCL C sources for the rtdevlib. They are
# compiled to SPIR-V binary and embedded into the CHIP
# library. <build-dir>/bitcode/rtdevlib-modules.h declares the
# embedded modules as 'std::array <basename-of-the-source>'.
#
# Use only characters allowed in C/C++/OpenCL-C language for the
# source names.
#
# Sources requiring SPIR-V 1.2 at most.
set(RTDEVLIB_SOURCES_v1_2
  atomicAddFloat_native atomicAddFloat_emulation
  atomicAddDouble_native atomicAddDouble_emulation
  atomicMinMaxFloat_emulation)

# Sources requiring SPIR-V 1.3 at most.
set(RTDEVLIB_SOURCES_v1_3
  ballot_native)

# Pick the chipStar-suffixed triple form on LLVM 22+ (in-tree SPIR-V
# backend); the simpler "spirvNN" form otherwise. Width is templated in
# per-width below.
function(rtdevlib_triple WIDTH MAX_SPIRV_VERSION OUT_VAR)
  if(LLVM_VERSION_MAJOR GREATER_EQUAL 22 AND CHIP_LLVM_USE_INTERGRATED_SPIRV)
    set(${OUT_VAR} "spirv${WIDTH}v${MAX_SPIRV_VERSION}-unknown-chipstar"
        PARENT_SCOPE)
  else()
    set(${OUT_VAR} "spirv${WIDTH}" PARENT_SCOPE)
  endif()
endfunction()

# Compile LLVM bitcode to SPIR-V binary which is then embedded into
# std::array<unsigned char, N> <ARRAY_NAME>.
function(embed_spirv_in_cpp
    ARRAY_NAME BC_SOURCE OUTPUT_SOURCE OUTPUT_HEADER MAX_SPIRV_VERSION)
  set(SPIRV_EXTENSIONS "+SPV_EXT_shader_atomic_float_add")
  # Per-output SPIR-V binary name (derived from the source basename rather
  # than the array name so the by-product path is per-width unique).
  get_filename_component(SOURCE_BASENAME "${BC_SOURCE}" NAME_WLE)
  set(SPIR_BINARY ${SOURCE_BASENAME}.spv)
  # Use the in-tree LLVM SPIR-V backend via clang when integrated SPIR-V is
  # enabled (LLVM 22+), or the external llvm-spirv translator otherwise.
  if(LLVM_VERSION_MAJOR GREATER_EQUAL 22 AND CHIP_LLVM_USE_INTERGRATED_SPIRV)
    # In-tree SPIR-V backend via clang -- the rtdevlib width is implied by
    # the bitcode's own datalayout.
    add_custom_command(
      OUTPUT "${OUTPUT_SOURCE}" "${OUTPUT_HEADER}"
      DEPENDS "${BC_SOURCE}"
      BYPRODUCTS "${SPIR_BINARY}"
      COMMAND "${CMAKE_CXX_COMPILER}"
      -mllvm -spirv-ext=${SPIRV_EXTENSIONS}
      -c -x ir
      "${BC_SOURCE}" -o "${SPIR_BINARY}"
      COMMAND ${CMAKE_SOURCE_DIR}/scripts/embed-binary-in-cpp.bash
      ${ARRAY_NAME} ${SPIR_BINARY} ${OUTPUT_SOURCE} ${OUTPUT_HEADER}
      COMMENT "Generating embedded SPIR-V binary (in-tree backend): ${OUTPUT_SOURCE}"
      VERBATIM
    )
  else()
    # External llvm-spirv translator
    add_custom_command(
      OUTPUT "${OUTPUT_SOURCE}" "${OUTPUT_HEADER}"
      DEPENDS "${BC_SOURCE}"
      BYPRODUCTS "${SPIR_BINARY}"
      COMMAND "${LLVM_SPIRV}"
      --spirv-ext=${SPIRV_EXTENSIONS}
      --spirv-max-version=${MAX_SPIRV_VERSION}
      "${BC_SOURCE}" -o "${SPIR_BINARY}"
      COMMAND ${CMAKE_SOURCE_DIR}/scripts/embed-binary-in-cpp.bash
      ${ARRAY_NAME} ${SPIR_BINARY} ${OUTPUT_SOURCE} ${OUTPUT_HEADER}
      COMMENT "Generating embedded SPIR-V binary (llvm-spirv): ${OUTPUT_SOURCE}"
      VERBATIM
    )
  endif()
endfunction()

# Phase 3 of chipstar_opencl_32bit proposal: produce rtdevlib SPIR-V
# modules per enabled pointer width. The *primary* width (first entry in
# CHIP_TARGET_POINTER_WIDTHS) is published under the un-suffixed
# `chipstar::<source>` array name for backward compatibility with the
# Level0 backend and with single-width chipStar builds. Additional widths
# are published under `chipstar::<source>_wNN`. The OpenCL backend picks
# the right variant at module-link time based on CL_DEVICE_ADDRESS_BITS.
#
# Single-width default (CHIP_TARGET_POINTER_WIDTHS=64) is byte-identical
# to the pre-Phase-3 build for the OpenCL and Level0 backends.
list(GET CHIP_TARGET_POINTER_WIDTHS 0 _RTDEVLIB_PRIMARY_WIDTH)

# Helper invoked per (width, source, spv-version-tag). Outputs:
#   - .bc  (compiled from the .cl at <triple-for-width>)
#   - .spv (translated from .bc)
#   - .cc  (embedded byte array named <ARRAY_NAME>)
#   - .h   (extern declaration for <ARRAY_NAME>)
# Appends the .cc and .h paths to the parent-scope lists.
function(add_rtdevlib_variant WIDTH SOURCE MAX_SPIRV_VERSION ARRAY_NAME)
  rtdevlib_triple(${WIDTH} ${MAX_SPIRV_VERSION} _triple)
  set(_bc "${CMAKE_CURRENT_BINARY_DIR}/${SOURCE}_w${WIDTH}.bc")
  set(_cc "${CMAKE_CURRENT_BINARY_DIR}/${ARRAY_NAME}.cc")
  set(_h  "${CMAKE_CURRENT_BINARY_DIR}/${ARRAY_NAME}.h")

  add_opencl_bitcode(
    "${_triple}"
    "${CMAKE_CURRENT_SOURCE_DIR}/${SOURCE}.cl"
    "${_bc}")
  embed_spirv_in_cpp(
    "${ARRAY_NAME}"
    "${_bc}"
    "${_cc}"
    "${_h}"
    "${MAX_SPIRV_VERSION}")

  set(RTDEVLIB_BITCODES "${RTDEVLIB_BITCODES};${_bc}" PARENT_SCOPE)
  set(RTDEVLIB_SOURCES_GENERATED "${RTDEVLIB_SOURCES_GENERATED};${_cc}" PARENT_SCOPE)
  set(RTDEVLIB_HEADERS "${RTDEVLIB_HEADERS};${_h}" PARENT_SCOPE)
endfunction()

set(RTDEVLIB_BITCODES "")
set(RTDEVLIB_SOURCES_GENERATED "")
set(RTDEVLIB_HEADERS "")

# v1.2 modules.
foreach(SOURCE IN LISTS RTDEVLIB_SOURCES_v1_2)
  foreach(WIDTH IN LISTS CHIP_TARGET_POINTER_WIDTHS)
    if(WIDTH STREQUAL _RTDEVLIB_PRIMARY_WIDTH)
      set(_array_name "${SOURCE}")
    else()
      set(_array_name "${SOURCE}_w${WIDTH}")
    endif()
    add_rtdevlib_variant("${WIDTH}" "${SOURCE}" "1.2" "${_array_name}")
  endforeach()
endforeach()

# v1.3 modules.
foreach(SOURCE IN LISTS RTDEVLIB_SOURCES_v1_3)
  foreach(WIDTH IN LISTS CHIP_TARGET_POINTER_WIDTHS)
    if(WIDTH STREQUAL _RTDEVLIB_PRIMARY_WIDTH)
      set(_array_name "${SOURCE}")
    else()
      set(_array_name "${SOURCE}_w${WIDTH}")
    endif()
    add_rtdevlib_variant("${WIDTH}" "${SOURCE}" "1.3" "${_array_name}")
  endforeach()
endforeach()

add_custom_target("rtdevlib-bitcodes" DEPENDS ${RTDEVLIB_BITCODES})

# Generate header that declares all the arrays of the embedded SPIR-V binaries.
# Also defines compile-time macros so the OpenCL backend can dispatch the
# right per-width variant at runtime via a single CHIPSTAR_RTDEVLIB_PICK
# macro (see uses in CHIPBackendOpenCL.cc).
set(RTDEVLIB_HEADER "${CMAKE_CURRENT_BINARY_DIR}/rtdevlib-modules.h")
string(REPLACE ";" " " RTDEVLIB_HEADERS_WITH_SPACES "${RTDEVLIB_HEADERS}")

# Compute the secondary width (if any). Single-width builds leave this
# unset; multi-width builds set it to the *non-primary* width.
set(_RTDEVLIB_SECONDARY_WIDTH "")
foreach(_w IN LISTS CHIP_TARGET_POINTER_WIDTHS)
  if(NOT _w STREQUAL _RTDEVLIB_PRIMARY_WIDTH)
    set(_RTDEVLIB_SECONDARY_WIDTH "${_w}")
  endif()
endforeach()

# Write the rtdevlib-modules prelude (preprocessor definitions and the
# CHIPSTAR_RTDEVLIB_PICK dispatch macro) at configure time. file(WRITE)
# preserves the literal newlines that the C preprocessor needs to see
# inside the multi-line macro definition -- attempting this via a bash
# heredoc or printf %b inside add_custom_command runs afoul of CMake's
# escaping of the COMMAND argument.
set(_RTDEVLIB_PRELUDE_FILE "${CMAKE_CURRENT_BINARY_DIR}/rtdevlib-prelude.h")
set(_PRELUDE_BASE "// Auto-generated by chipStar bitcode/CMakeLists.txt.
#pragma once
#include <array>
#include <cstddef>
#include <utility>
#define CHIPSTAR_RTDEVLIB_PRIMARY_WIDTH ${_RTDEVLIB_PRIMARY_WIDTH}
")
if(_RTDEVLIB_SECONDARY_WIDTH STREQUAL "")
  file(WRITE "${_RTDEVLIB_PRELUDE_FILE}"
"${_PRELUDE_BASE}#define CHIPSTAR_RTDEVLIB_PICK(NAME, ADDR_BITS) \\
  ::std::make_pair((const unsigned char*)::chipstar::NAME.data(), ::chipstar::NAME.size())
")
else()
  file(WRITE "${_RTDEVLIB_PRELUDE_FILE}"
"${_PRELUDE_BASE}#define CHIPSTAR_RTDEVLIB_SECONDARY_WIDTH ${_RTDEVLIB_SECONDARY_WIDTH}
#define CHIPSTAR_RTDEVLIB_SECONDARY_SUFFIX _w${_RTDEVLIB_SECONDARY_WIDTH}
#define CHIPSTAR_RTDEVLIB_TOKENPASTE_(a, b) a##b
#define CHIPSTAR_RTDEVLIB_TOKENPASTE(a, b) CHIPSTAR_RTDEVLIB_TOKENPASTE_(a, b)
#define CHIPSTAR_RTDEVLIB_PICK(NAME, ADDR_BITS) \\
  (((ADDR_BITS) == CHIPSTAR_RTDEVLIB_PRIMARY_WIDTH) \\
    ? ::std::make_pair((const unsigned char*)::chipstar::NAME.data(), ::chipstar::NAME.size()) \\
    : ::std::make_pair((const unsigned char*)::chipstar::CHIPSTAR_RTDEVLIB_TOKENPASTE(NAME, CHIPSTAR_RTDEVLIB_SECONDARY_SUFFIX).data(), \\
                       ::chipstar::CHIPSTAR_RTDEVLIB_TOKENPASTE(NAME, CHIPSTAR_RTDEVLIB_SECONDARY_SUFFIX).size()))
")
endif()

add_custom_command(
  OUTPUT ${RTDEVLIB_HEADER}
  DEPENDS ${RTDEVLIB_HEADERS} "${_RTDEVLIB_PRELUDE_FILE}"
  COMMAND bash -c "cat ${_RTDEVLIB_PRELUDE_FILE} ${RTDEVLIB_HEADERS_WITH_SPACES} > ${RTDEVLIB_HEADER}"
  COMMENT "Generating rtdevlib header (primary=${_RTDEVLIB_PRIMARY_WIDTH}, secondary=${_RTDEVLIB_SECONDARY_WIDTH})."
  VERBATIM
)

add_custom_target(rtdevlib-header DEPENDS ${RTDEVLIB_HEADER})

add_library(rtdevlib OBJECT
  ${RTDEVLIB_SOURCES_GENERATED} ${RTDEVLIB_HEADER})
