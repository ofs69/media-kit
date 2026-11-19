# This file is a part of media_kit (https://github.com/media-kit/media-kit).
#
# Copyright © 2021 & onwards, Hitesh Kumar Saini <saini123hitesh@gmail.com>.
# All rights reserved.
# Use of this source code is governed by MIT license that can be found in the LICENSE file.

cmake_minimum_required(VERSION 3.14)
cmake_policy(SET CMP0175 NEW)

# This option is read by the other packages which are part of package:media_kit.
option(MEDIA_KIT_LIBS_AVAILABLE "package:media_kit libraries are available." ON)

set(PROJECT_NAME "media_kit_libs_windows_video")
project(${PROJECT_NAME} LANGUAGES CXX)

# Deal with MSVC incompatiblity
add_compile_definitions(_DISABLE_CONSTEXPR_MUTEX_CONSTRUCTOR)

# ------------------------------------------------------------------------------
function(download_and_verify url algorithm checksum archive)
  if(EXISTS "${archive}")
    file(${algorithm} "${archive}" archive_checksum)
    if(archive_checksum STREQUAL checksum)
      return()
    endif()
  endif()

  message(STATUS "Downloading ${url}")
  file(DOWNLOAD "${url}" "${archive}.part"
    EXPECTED_HASH "${algorithm}=${checksum}"
    TLS_VERIFY ON
  )
  file(RENAME "${archive}.part" "${archive}")
endfunction()

function(check_directory_exists_and_not_empty dir result_var)
  # Check if the directory exists
  if(EXISTS "${dir}")
    # Check if the directory is not empty
    file(GLOB dir_contents "${dir}/*")

    if(dir_contents)
      set(${result_var} TRUE PARENT_SCOPE)
    else()
      set(${result_var} FALSE PARENT_SCOPE)
      message(STATUS "Directory ${dir} exists but is empty!")
    endif()
  else()
    set(${result_var} FALSE PARENT_SCOPE)
    message(STATUS "Directory ${dir} does not exist!")
  endif()
endfunction()

# This build includes mpv's EGL_EXT_device_query fix for D3D11/ANGLE interop.
if(FLUTTER_TARGET_PLATFORM STREQUAL "windows-x64")
  set(LIBMPV_ARCH "x86_64")
  set(LIBMPV_SHA256 "7c147cb7ceaba717414e782ef4398de728f679db6e86daaa8aeb175670cb7994")
elseif(FLUTTER_TARGET_PLATFORM STREQUAL "windows-arm64")
  set(LIBMPV_ARCH "aarch64")
  set(LIBMPV_SHA256 "f7d3b2d97d07b5758cb277412ed95dc55aac2112397b6d06161d0b58c3e941fc")
else()
  message(FATAL_ERROR "Unsupported Windows target: ${FLUTTER_TARGET_PLATFORM}")
endif()

set(LIBMPV "mpv-dev-${LIBMPV_ARCH}-20261108-git-55d3feb45c")
set(LIBMPV_ARCHIVE "${CMAKE_BINARY_DIR}/${LIBMPV}.7z")
download_and_verify(
  "https://downloads.sourceforge.net/project/mpv-player-windows/libmpv/${LIBMPV}.7z"
  SHA256 "${LIBMPV_SHA256}" "${LIBMPV_ARCHIVE}"
)

# Version the extraction directory so incremental builds link and ship the
# same library as clean builds. Only mark a complete extraction as reusable.
set(LIBMPV_SRC "${CMAKE_CURRENT_BINARY_DIR}/${LIBMPV}")
if(NOT EXISTS "${LIBMPV_SRC}/.extracted")
  file(ARCHIVE_EXTRACT INPUT "${LIBMPV_ARCHIVE}" DESTINATION "${LIBMPV_SRC}")
  file(TOUCH "${LIBMPV_SRC}/.extracted")
endif()
set(MEDIA_KIT_LIBMPV_DIR "${LIBMPV_SRC}" PARENT_SCOPE)

find_program(SEVENZ_EXECUTABLE NAMES 7z 7za)

# ------------------------------------------------------------------------------
# ANGLE archive containing the pre-built shared libraries & headers.
set(ANGLE "ANGLE.7z")

# Download URL & MD5 hash of the ANGLE archive.
set(ANGLE_URL "https://github.com/alexmercerind/flutter-windows-ANGLE-OpenGL-ES/releases/download/v1.0.1/ANGLE.7z")
set(ANGLE_MD5 "e866f13e8d552348058afaafe869b1ed")

# Download location of the ANGLE archive.
set(ANGLE_ARCHIVE "${CMAKE_BINARY_DIR}/${ANGLE}")
set(ANGLE_SRC "${CMAKE_BINARY_DIR}/ANGLE")

download_and_verify(
  ${ANGLE_URL}
  MD5 ${ANGLE_MD5}
  ${ANGLE_ARCHIVE}
)

check_directory_exists_and_not_empty(${ANGLE_SRC} ANGLE_SRC_VALID)

# Extract the ANGLE archive.
if(NOT ANGLE_SRC_VALID)
  message(STATUS "Extracting ${ANGLE}...")
  make_directory("${ANGLE_SRC}")
  add_custom_target("${PROJECT_NAME}_ANGLE_EXTRACT" ALL)
  
  if(SEVENZ_EXECUTABLE)
    message(STATUS "Using 7zip for extraction: ${SEVENZ_EXECUTABLE}")
    add_custom_command(
      TARGET "${PROJECT_NAME}_ANGLE_EXTRACT"
      POST_BUILD
      COMMAND "${SEVENZ_EXECUTABLE}" x -y "\"${ANGLE_ARCHIVE}\""
      WORKING_DIRECTORY "${ANGLE_SRC}"
    )
  else()
    message(STATUS "7zip not found, falling back to CMake tar")
    add_custom_command(
      TARGET "${PROJECT_NAME}_ANGLE_EXTRACT"
      COMMAND "${CMAKE_COMMAND}" -E tar xzf "\"${ANGLE_ARCHIVE}\""
      WORKING_DIRECTORY "${ANGLE_SRC}"
    )
  endif()
endif()

# ------------------------------------------------------------------------------
set(PLUGIN_NAME "media_kit_libs_windows_video_plugin")

add_library(
  ${PLUGIN_NAME} SHARED
  "include/media_kit_libs_windows_video/media_kit_libs_windows_video_plugin_c_api.h"
  "media_kit_libs_windows_video_plugin_c_api.cpp"
)

apply_standard_settings(${PLUGIN_NAME})

set_target_properties(
  ${PLUGIN_NAME}
  PROPERTIES
  CXX_VISIBILITY_PRESET
  hidden
)
target_compile_definitions(
  ${PLUGIN_NAME}
  PRIVATE
  FLUTTER_PLUGIN_IMPL
)

target_include_directories(
  ${PLUGIN_NAME} INTERFACE
  "${CMAKE_CURRENT_SOURCE_DIR}/include"
)
target_link_libraries(
  ${PLUGIN_NAME}
  PRIVATE
  flutter
  flutter_wrapper_plugin
)

set(
  media_kit_libs_windows_video_bundled_libraries
  "${LIBMPV_SRC}/libmpv-2.dll"
  # "${ANGLE_SRC}/d3dcompiler_47.dll"
  "${ANGLE_SRC}/libEGL.dll"
  "${ANGLE_SRC}/libGLESv2.dll"
  # "${ANGLE_SRC}/vk_swiftshader.dll"
  # "${ANGLE_SRC}/vulkan-1.dll"
  # "${ANGLE_SRC}/zlib.dll"
  PARENT_SCOPE
)
