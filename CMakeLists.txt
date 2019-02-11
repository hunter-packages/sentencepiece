# Copyright 2018 Google Inc.
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
# limitations under the License.!

cmake_minimum_required(VERSION 3.2 FATAL_ERROR)

include("cmake/HunterGate.cmake")
HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.23.101.tar.gz"
    SHA1 "ccb234f5ea9dc89989b28164cb38cb4ebb04237c"
)

file(STRINGS "VERSION" SPM_VERSION)
message(STATUS "VERSION: ${SPM_VERSION}")
project(sentencepiece VERSION ${SPM_VERSION})

option(SPM_ENABLE_NFKC_COMPILE "Enables NFKC compile" OFF)
option(SPM_ENABLE_SHARED "Builds shared libaries in addition to static libraries." ON)
option(SPM_BUILD_TEST "Builds test binaries." OFF)
option(SPM_COVERAGE "Runs gcov to test coverage." OFF)
option(SPM_ENABLE_TENSORFLOW_SHARED "Makes a tensorflow compatible shared file." OFF)
option(SPM_ENABLE_TCMALLOC "Enable TCMalloc if available." ON)
option(SPM_TCMALLOC_STATIC "Link static library of TCMALLOC." OFF)
option(SPM_NO_THREADLOCAL "Disable thread_local operator" OFF)
option(SPM_USE_BUILTIN_PROTOBUF "Use built-in protobuf" ON)
option(SPM_BUILD_TOOLS "Install tools" OFF)

set(CMAKE_CXX_STANDARD 11 CACHE STRING "Default C++ standard")
set(CMAKE_CXX_STANDARD_REQUIRED ON CACHE BOOL "")

set(prefix ${CMAKE_INSTALL_PREFIX})
set(exec_prefix "\${prefix}")
set(libdir "\${exec_prefix}/lib")
set(includedir "\${prefix}/include")
set(GNUCXX_STD_SUPPORT_VERSION "4.3")

if(HUNTER_ENABLED)
  # If we cross compile for Android or iOS build a separate protoc executable on host to compile .proto files in CMake
  if(IOS OR ANDROID)
    include(hunter_experimental_add_host_project)
    hunter_experimental_add_host_project(cmake/host)

    add_executable(protobuf::protoc IMPORTED)
    set(Protobuf_PROTOC_EXECUTABLE "${HUNTER_HOST_ROOT}/bin/protoc")
    set_target_properties(
        protobuf::protoc
        PROPERTIES
        IMPORTED_LOCATION
        "${Protobuf_PROTOC_EXECUTABLE}"
    )
    set_target_properties(
        protobuf::protoc
        PROPERTIES
        IMPORTED_LOCATION_RELEASE
        "${Protobuf_PROTOC_EXECUTABLE}"
    )
  endif()

  option(protobuf_MODULE_COMPATIBLE "use protobuf in module compatible mode" ON)
  hunter_add_package(Protobuf)
  find_package(Protobuf CONFIG REQUIRED)
  set(libprotobuf_lite protobuf::libprotobuf-lite)
else()

if (SPM_USE_BUILTIN_PROTOBUF)
  set(libprotobuf_lite "")
else()
  set(libprotobuf_lite "-lprotobuf-lite")
endif()

endif()

if (MSVC)
  string(REPLACE "/MD" "/MT" CMAKE_CXX_FLAGS_DEBUG          ${CMAKE_CXX_FLAGS_DEBUG})
  string(REPLACE "/MD" "/MT" CMAKE_CXX_FLAGS_MINSIZEREL     ${CMAKE_CXX_FLAGS_MINSIZEREL})
  string(REPLACE "/MD" "/MT" CMAKE_CXX_FLAGS_RELEASE        ${CMAKE_CXX_FLAGS_RELEASE})
  string(REPLACE "/MD" "/MT" CMAKE_CXX_FLAGS_RELWITHDEBINFO ${CMAKE_CXX_FLAGS_RELWITHDEBINFO})
  add_definitions("/wd4267 /wd4244 /wd4305 /Zc:strictStrings /utf-8")
endif()

if (APPLE)
  set(CMAKE_MACOSX_RPATH ON)
  set(CMAKE_SKIP_BUILD_RPATH FALSE)
  set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
  set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
  set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
  list(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/lib" isSystemDir)
  if ("${isSystemDir}" STREQUAL "-1")
    set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
  endif()
endif()

if (NOT DEFINED CMAKE_INSTALL_BINDIR)
  set(CMAKE_INSTALL_BINDIR bin)
endif()

if (NOT DEFINED CMAKE_INSTALL_LIBDIR)
  set(CMAKE_INSTALL_LIBDIR lib)
endif()

if (NOT DEFINED CMAKE_INSTALL_LIBDIR)
  set(CMAKE_INSTALL_LIBDIR lib)
endif()

if (NOT DEFINED CMAKE_INSTALL_INCDIR)
  set(CMAKE_INSTALL_INCDIR include)
endif()

configure_file("${PROJECT_SOURCE_DIR}/config.h.in" "config.h")
configure_file("${PROJECT_SOURCE_DIR}/sentencepiece.pc.in" "sentencepiece.pc" @ONLY)

if (NOT MSVC)
  install(FILES "${CMAKE_BINARY_DIR}/sentencepiece.pc" DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
endif()

include_directories(${CMAKE_SOURCE_DIR} ${PROJECT_BINARY_DIR})

if (SPM_BUILD_TEST)
  enable_testing()
endif()

### Installation (https://github.com/forexample/package-example) {

# Introduce variables:
# * CMAKE_INSTALL_LIBDIR
# * CMAKE_INSTALL_BINDIR
# * CMAKE_INSTALL_INCLUDEDIR
include(GNUInstallDirs)

set(config_install_dir "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}")
set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")
set(NAMESPACE "${PROJECT_NAME}::")

include(CMakePackageConfigHelpers)

# Use:
# * PROJECT_VERSION
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

# Use:
# * TARGETS_EXPORT_NAME
# * PROJECT_NAME
configure_package_config_file(
    "cmake/template/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

install(
    FILES "${project_config}" "${version_config}"
    DESTINATION "${config_install_dir}"
)

install(
    EXPORT "${TARGETS_EXPORT_NAME}"
    NAMESPACE "${NAMESPACE}"
    DESTINATION "${config_install_dir}"
)

### }

add_subdirectory(src)
add_subdirectory(third_party)

set(CPACK_SOURCE_GENERATOR "TXZ")
set(CPACK_GENERATOR "7Z")
set(CPACK_PACKAGE_VERSION "${SPM_VERSION}")
set(CPACK_STRIP_FILES TRUE)
set(CPACK_RESOURCE_FILE_LICENSE "${PROJECT_SOURCE_DIR}/LICENSE")
set(CPACK_RESOURCE_FILE_README "${PROJECT_SOURCE_DIR}/README.md")
set(CPACK_PACKAGE_CONTACT "taku@google.com")
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Taku Kudo")
set(CPACK_SOURCE_IGNORE_FILES "/build/;/.git/;/dist/;/sdist/;~$;${CPACK_SOURCE_IGNORE_FILES}")
include(CPack)
