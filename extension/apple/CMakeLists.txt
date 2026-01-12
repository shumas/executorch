# Copyright (c) Meta Platforms, Inc. and affiliates.
# All rights reserved.
#
# This source code is licensed under the BSD-style license found in the
# LICENSE file in the root directory of this source tree.

# Please this file formatted by running:
# ~~~
# cmake-format -i CMakeLists.txt
# ~~~

cmake_minimum_required(VERSION 3.19)

enable_language(Swift)

# Source root directory for executorch.
if(NOT EXECUTORCH_ROOT)
  set(EXECUTORCH_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/../..)
endif()

add_library(extension_apple)

file(GLOB OBJC_SOURCES ExecuTorch/Exported/*.m ExecuTorch/Exported/*.mm
     ExecuTorch/Internal/*.m ExecuTorch/Internal/*.mm
)

file(GLOB SWIFT_SOURCES ExecuTorch/Exported/*.swift)

target_sources(extension_apple PRIVATE ${OBJC_SOURCES} ${SWIFT_SOURCES})

target_include_directories(
  extension_apple
  PUBLIC ExecuTorch/Exported
  PRIVATE ExecuTorch/Internal
)

find_library(FOUNDATION_FRAMEWORK Foundation)
target_link_libraries(
  extension_apple PRIVATE executorch ${FOUNDATION_FRAMEWORK}
)

set_source_files_properties(
  ${OBJC_SOURCES} PROPERTIES COMPILE_FLAGS "-fobjc-arc" "-fno-exceptions"
                                                        "-fno-rtti"
)

set(MODULE_MAP_DIR ${CMAKE_CURRENT_BINARY_DIR}/module)
set(MODULE_MAP_FILE ${MODULE_MAP_DIR}/module.modulemap)

configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/ExecuTorch/Exported/ExecuTorch.h"
  "${MODULE_MAP_DIR}/ExecuTorch.h" COPYONLY
)

file(MAKE_DIRECTORY ${MODULE_MAP_DIR})
file(
  WRITE ${MODULE_MAP_FILE}
  "module ExecuTorch {
  umbrella header \"ExecuTorch.h\"
  export *
}
"
)

set(SWIFT_CLANG_INTEROP_FLAGS
    "-Xcc -fmodule-map-file=${MODULE_MAP_FILE} -I ${MODULE_MAP_DIR}"
)
set(SWIFT_REMAP_FLAGS "-debug-prefix-map ${PROJECT_SOURCE_DIR}=/executorch")

if(NOT CMAKE_GENERATOR MATCHES "Xcode")
  target_compile_options(
    extension_apple
    PRIVATE
      $<$<COMPILE_LANGUAGE:Swift>:-Xcc>
      $<$<COMPILE_LANGUAGE:Swift>:-fmodule-map-file=${MODULE_MAP_FILE}>
      $<$<COMPILE_LANGUAGE:Swift>:-Xcc>
      $<$<COMPILE_LANGUAGE:Swift>:-I${MODULE_MAP_DIR}>
  )
  if(DEFINED CMAKE_C_COMPILER_TARGET)
    target_compile_options(
      extension_apple
      PRIVATE
        $<$<COMPILE_LANGUAGE:Swift>:-target>
        $<$<COMPILE_LANGUAGE:Swift>:${CMAKE_C_COMPILER_TARGET}>
    )
  endif()
  set(_swift_interface_dir "${CMAKE_Swift_MODULE_DIRECTORY}")
  if(NOT _swift_interface_dir)
    set(_swift_interface_dir "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}")
  endif()
  if(NOT _swift_interface_dir)
    set(_swift_interface_dir "${CMAKE_CURRENT_BINARY_DIR}")
  endif()
  file(MAKE_DIRECTORY "${_swift_interface_dir}")
  if(DEFINED CMAKE_C_COMPILER_TARGET)
    set(_swift_interface_file
        "${_swift_interface_dir}/${CMAKE_C_COMPILER_TARGET}.swiftinterface"
    )
  else()
    set(_swift_interface_file "${_swift_interface_dir}/ExecuTorch.swiftinterface")
  endif()
  target_compile_options(
    extension_apple
    PRIVATE
      $<$<COMPILE_LANGUAGE:Swift>:-enable-library-evolution>
      $<$<COMPILE_LANGUAGE:Swift>:-emit-module-interface>
      $<$<COMPILE_LANGUAGE:Swift>:-emit-module-interface-path>
      $<$<COMPILE_LANGUAGE:Swift>:${_swift_interface_file}>
  )
endif()

set_target_properties(
  extension_apple
  PROPERTIES Swift_MODULE_NAME "ExecuTorch"
             Swift_FLAGS "${SWIFT_CLANG_INTEROP_FLAGS} ${SWIFT_REMAP_FLAGS}"
             XCODE_ATTRIBUTE_SWIFT_MODULE_NAME "ExecuTorch"
             XCODE_ATTRIBUTE_BUILD_LIBRARY_FOR_DISTRIBUTION "YES"
             XCODE_ATTRIBUTE_OTHER_SWIFT_FLAGS
             "${SWIFT_CLANG_INTEROP_FLAGS} ${SWIFT_REMAP_FLAGS}"
)

add_custom_command(
  TARGET extension_apple
  POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E rm -rf ${MODULE_MAP_DIR}
)
