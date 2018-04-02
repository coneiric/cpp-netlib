# Copyright (c) Glyn Matthews 2011.
# Copyright 2011 Dean Michael Berris (dberris@google.com)
# Copyright 2011 Google, Inc.
# Distributed under the Boost Software License, Version 1.0.
#    (See accompanying file LICENSE_1_0.txt or copy at
#          http://www.boost.org/LICENSE_1_0.txt)

cmake_minimum_required(VERSION 3.5 FATAL_ERROR)
project(cppnetlib-libs CXX)

file(GLOB_RECURSE CPP-NETLIB_HEADERS
    "${CPP-NETLIB_SOURCE_DIR}/boost/" "*.hpp")

add_library(uri uri/uri.cpp uri/schemes.cpp)
target_compile_definitions(uri PRIVATE BOOST_SPIRIT_THREADSAFE)
target_link_libraries(uri PUBLIC cppnetlib Boost::system)
set_target_properties(uri
    PROPERTIES VERSION ${CPPNETLIB_VERSION_STRING}
    SOVERSION ${CPPNETLIB_VERSION_MAJOR}
    PUBLIC_HEADER "${CPP-NETLIB_HEADERS}")
install(TARGETS uri 
    EXPORT cppnetlibTargets 
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_FULL_INCLUDEDIR} 
    LIBRARY DESTINATION ${CMAKE_INSTALL_FULL_LIBDIR} 
    ARCHIVE DESTINATION ${CMAKE_INSTALL_FULL_LIBDIR}) 

add_library(server-parsers server_request_parsers_impl.cpp)
target_link_libraries(server-parsers PUBLIC cppnetlib)
if (TARGET OpenSSL::SSL)
  target_link_libraries(server-parsers PUBLIC OpenSSL::SSL)
endif()
set_target_properties(server-parsers
    PROPERTIES VERSION ${CPPNETLIB_VERSION_STRING}
    SOVERSION ${CPPNETLIB_VERSION_MAJOR}
    PUBLIC_HEADER "${CPP-NETLIB_HEADERS}")
install(TARGETS server-parsers 
    EXPORT cppnetlibTargets 
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_FULL_INCLUDEDIR} 
    LIBRARY DESTINATION ${CMAKE_INSTALL_FULL_LIBDIR} 
    ARCHIVE DESTINATION ${CMAKE_INSTALL_FULL_LIBDIR}) 

add_library(client-connections client.cpp)
target_link_libraries(client-connections PUBLIC cppnetlib Boost::regex)
if (TARGET OpenSSL::SSL)
  target_link_libraries(client-connections PUBLIC OpenSSL::SSL)
endif()
set_target_properties(client-connections
  PROPERTIES VERSION ${CPPNETLIB_VERSION_STRING}
  SOVERSION ${CPPNETLIB_VERSION_MAJOR}
  PUBLIC_HEADER "${CPP-NETLIB_HEADERS}")
install(TARGETS client-connections 
    EXPORT cppnetlibTargets 
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_FULL_INCLUDEDIR} 
    LIBRARY DESTINATION ${CMAKE_INSTALL_FULL_LIBDIR} 
    ARCHIVE DESTINATION ${CMAKE_INSTALL_FULL_LIBDIR}) 