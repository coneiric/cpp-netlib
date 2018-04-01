# Copyright (c) Dean Michael Berris 2010.
# Copyright 2016 Google, Inc.
# Distributed under the Boost Software License, Version 1.0.
# (See accompanying file LICENSE_1_0.txt or copy at
# http://www.boost.org/LICENSE_1_0.txt)

cmake_minimum_required(VERSION 3.5 FATAL_ERROR)
project(CPP-NETLIB CXX)

option( CPP-NETLIB_BUILD_SHARED_LIBS "Build cpp-netlib as shared libraries." OFF )
option( CPP-NETLIB_BUILD_TESTS "Build the cpp-netlib project tests." ON)
# option( CPP-NETLIB_BUILD_EXPERIMENTS "Build the cpp-netlib project experiments." ON)
option( CPP-NETLIB_BUILD_EXAMPLES "Build the cpp-netlib project examples." ON)
option( CPP-NETLIB_ENABLE_HTTPS "Build cpp-netlib with support for https if OpenSSL is found." ON)
option( CPP-NETLIB_STATIC_OPENSSL "Build cpp-netlib using static OpenSSL" OFF)
option( CPP-NETLIB_STATIC_BOOST "Build cpp-netlib using static Boost" OFF)

if (NOT DEFINED CPP-NETLIB_WINAPI_VERSION)
  set(CPP-NETLIB_WINAPI_VERSION 0x0501)
endif()

include(GNUInstallDirs)

# determine install path for CMake config files
if(WIN32 AND NOT CYGWIN)
    set(DEF_INSTALL_CMAKE_DIR CMake)
else()
    set(DEF_INSTALL_CMAKE_DIR ${CMAKE_INSTALL_LIBDIR}/cmake/cppnetlib)
endif()
set(INSTALL_CMAKE_DIR ${DEF_INSTALL_CMAKE_DIR} CACHE PATH "Installation directory for CMake files")

# Make relative cmake install path absolute (needed later on)
if(NOT IS_ABSOLUTE "${INSTALL_CMAKE_DIR}")
    set(INSTALL_CMAKE_DIR "${CMAKE_INSTALL_PREFIX}/${INSTALL_CMAKE_DIR}")
endif()

if(CPP-NETLIB_BUILD_SHARED_LIBS OR BUILD_SHARED_LIBS)
  message (STATUS "Linking boost testing libs dynamically...")
  set(CPP-NETLIB_BUILD_SHARED_LIBS ON)
  set(BUILD_SHARED_LIBS ON)
else()
  set(CPP-NETLIB_BUILD_SHARED_LIBS OFF)
  set(BUILD_SHARED_LIBS OFF)
endif()

# Use Boost's static libraries
if (CPP-NETLIB_STATIC_BOOST)
  set(Boost_USE_STATIC_LIBS ON)
else()
  # We need this for all tests to use the dynamic version.
  set_property(DIRECTORY APPEND PROPERTY COMPILE_DEFINITIONS BOOST_ALL_DYN_LINK)
endif()

# The interface target for public usage requirements
add_library(cppnetlib INTERFACE)

# Always use multi-threaded Boost libraries.
set(Boost_USE_MULTITHREADED ON)

find_package(Boost 1.58.0 REQUIRED COMPONENTS system thread regex)

if (CPP-NETLIB_ENABLE_HTTPS)
 if (APPLE)
   # If we're on OS X check for Homebrew's copy of OpenSSL instead of Apple's
   if (NOT OpenSSL_DIR)
     find_program(HOMEBREW brew)
     if (NOT HOMEBREW)
       message(WARNING "Homebrew not found: not using Homebrew's OpenSSL")
       if (NOT OPENSSL_ROOT_DIR)
         message(WARNING "Use -DOPENSSL_ROOT_DIR for non-Apple OpenSSL")
       endif()
     else()
       execute_process(COMMAND brew --prefix openssl
         OUTPUT_VARIABLE OPENSSL_ROOT_DIR
         OUTPUT_STRIP_TRAILING_WHITESPACE)
     endif()
   endif()
 endif()

 if (CPP-NETLIB_STATIC_OPENSSL)
   set(OPENSSL_USE_STATIC_LIBS TRUE)
 endif()

 find_package(OpenSSL REQUIRED)
 target_compile_definitions(cppnetlib INTERFACE BOOST_NETWORK_ENABLE_HTTPS)
 if (CPP-NETLIB_STATIC_OPENSSL)
   target_link_libraries(cppnetlib INTERFACE ${CMAKE_DL_LIBS})
 endif()
endif()


set(CMAKE_VERBOSE_MAKEFILE true)

set(CPPNETLIB_VERSION_MAJOR 0) # MUST bump this whenever we make ABI-incompatible changes
set(CPPNETLIB_VERSION_MINOR 13)
set(CPPNETLIB_VERSION_PATCH 0)
set(CPPNETLIB_VERSION_STRING ${CPPNETLIB_VERSION_MAJOR}.${CPPNETLIB_VERSION_MINOR}.${CPPNETLIB_VERSION_PATCH})

set_property(DIRECTORY APPEND PROPERTY COMPILE_DEFINITIONS $<$<CONFIG:Debug>:BOOST_NETWORK_DEBUG>)

# Use C++11
if (CMAKE_VERSION VERSION_LESS 3.8)
  set(CMAKE_CXX_STANDARD 11)
  set(CMAKE_CXX_STANDARD_REQUIRED ON)
else()
  target_compile_features(cppnetlib INTERFACE cxx_std_11)
endif()

if (CMAKE_CXX_COMPILER_ID MATCHES "GNU")
  add_compile_options(-Wall)
elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  # A high enough template depth for the template metaprogramming.
  add_compile_options(-Wall -ftemplate-depth=256)
  if (CMAKE_SYSTEM_NAME MATCHES "Darwin")
    # Use libc++ only in OS X.
    add_compile_options(-stdlib=libc++)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -lc++") 
  endif()
endif()


if (WIN32)
  target_compile_definitions(cppnetlib INTERFACE _WIN32_WINNT=${CPP-NETLIB_WINAPI_VERSION} BOOST_USE_WINAPI_VERSION=${CPP-NETLIB_WINAPI_VERSION})

  if (MSVC)
    target_compile_options(cppnetlib INTERFACE /bigobj)
    target_compile_definitions(cppnetlib INTERFACE _SCL_SECURE_NO_WARNINGS)
  endif()

  if (MINGW)
    target_link_libraries(cppnetlib INTERFACE ws2_32 wsock32)
  endif()
endif()

target_include_directories(cppnetlib INTERFACE 
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

find_package(Threads REQUIRED)
target_link_libraries(cppnetlib INTERFACE Boost::thread Threads::Threads)

install(TARGETS cppnetlib EXPORT cppnetlibTargets)

add_subdirectory(libs/network/src)

add_library(cppnetlib::cppnetlib ALIAS cppnetlib)
add_library(cppnetlib::uri ALIAS uri)
add_library(cppnetlib::server-parsers ALIAS server-parsers)
add_library(cppnetlib::client-connections ALIAS client-connections)

if (CPP-NETLIB_BUILD_TESTS)
  enable_testing()

  if (MSVC)
    set(gtest_force_shared_crt ON CACHE BOOL "Use shared (DLL) run-time lib even when Google Test is built as static lib.")
  endif()

  add_subdirectory(deps/googletest)

  if (MSVC)
    target_compile_definitions(gtest PUBLIC _SILENCE_TR1_NAMESPACE_DEPRECATION_WARNING)
    target_compile_definitions(gtest_main PUBLIC _SILENCE_TR1_NAMESPACE_DEPRECATION_WARNING)
    target_compile_definitions(gmock PUBLIC _SILENCE_TR1_NAMESPACE_DEPRECATION_WARNING)
    target_compile_definitions(gmock_main PUBLIC _SILENCE_TR1_NAMESPACE_DEPRECATION_WARNING)
  endif()

  add_subdirectory(libs/network/test)
endif()

if (CPP-NETLIB_BUILD_EXAMPLES)
  add_library(cxxopts INTERFACE)
  target_include_directories(cxxopts INTERFACE deps/cxxopts/src)

  add_subdirectory(libs/network/example)
endif ()

install(DIRECTORY boost DESTINATION ${CMAKE_INSTALL_INCLUDEDIR} COMPONENT dev)

### 
## Export Targets 
# (so cpp-netlib can be easily used by other CMake projects) 
# [see http://www.cmake.org/Wiki/CMake/Tutorials/How_to_create_a_ProjectConfig.cmake_file] 
 
# Add all targets to the build-tree export set 
export(EXPORT cppnetlibTargets 
    FILE "${CMAKE_CURRENT_BINARY_DIR}/cppnetlibTargets.cmake"
    NAMESPACE cppnetlib::)
configure_file(cppnetlibConfig.cmake.in
    "${CMAKE_CURRENT_BINARY_DIR}/cppnetlibConfig.cmake" @ONLY)

# Export the package for use from the build-tree 
# (this registers the build-tree with a global CMake-registry) 
export(PACKAGE cppnetlib) 

include(CMakePackageConfigHelpers)
write_basic_package_version_file(cppnetlibConfigVersion.cmake VERSION ${CPPNETLIB_VERSION_STRING} COMPATIBILITY AnyNewerVersion)

# Install the cppnetlibConfig.cmake and cppnetlibConfigVersion.cmake 
install(FILES
    "${PROJECT_BINARY_DIR}/cppnetlibConfig.cmake" 
    "${PROJECT_BINARY_DIR}/cppnetlibConfigVersion.cmake" 
    DESTINATION "${INSTALL_CMAKE_DIR}"
    COMPONENT dev) 

# Install the export set for use with the install-tree 
install(EXPORT cppnetlibTargets 
    FILE cppnetlibTargets.cmake
    NAMESPACE cppnetlib::
    DESTINATION "${INSTALL_CMAKE_DIR}")