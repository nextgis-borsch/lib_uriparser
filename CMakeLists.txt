################################################################################
# Project:  Lib UriParser
# Purpose:  CMake build scripts
# Author:   Dmitry Baryshnikov, dmitry.baryshnikov@nexgis.com
################################################################################
# Copyright (C) 2018, NextGIS <info@nextgis.com>
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and/or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
# OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
# DEALINGS IN THE SOFTWARE.
################################################################################

cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)

project(uriparser)

if(NOT DEFINED PACKAGE_VENDOR)
    set(PACKAGE_VENDOR NextGIS)
endif()

if(NOT DEFINED PACKAGE_BUGREPORT)
    set(PACKAGE_BUGREPORT info@nextgis.com)
endif()

# Some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# Set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

if(OSX_FRAMEWORK AND (BUILD_SHARED_LIBS OR BUILD_STATIC_LIBS))
  message(FATAL_ERROR "Only OSX_FRAMEWORK key or any or both BUILD_SHARED_LIBS
                       and BUILD_STATIC_LIBS keys are permitted")
endif()

if(OSX_FRAMEWORK)
    set(INSTALL_BIN_DIR "bin" CACHE INTERNAL "Installation directory for executables" FORCE)
    set(INSTALL_LIB_DIR "Library/Frameworks" CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(INSTALL_INC_DIR ${INSTALL_LIB_DIR}/${PROJECT_NAME}.framework/Headers CACHE INTERNAL "Installation directory for headers" FORCE)
    set(INSTALL_CMAKECONF_DIR ${INSTALL_LIB_DIR}/${PROJECT_NAME}.framework/Resources/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
    set(SKIP_INSTALL_HEADERS ON)
    set(SKIP_INSTALL_FILES ON)
    set(SKIP_INSTALL_EXPORT ON)
    set(CMAKE_MACOSX_RPATH ON)
else()
    include(GNUInstallDirs)

    set(INSTALL_BIN_DIR ${CMAKE_INSTALL_BINDIR} CACHE INTERNAL "Installation directory for executables" FORCE)
    set(INSTALL_LIB_DIR ${CMAKE_INSTALL_LIBDIR} CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(INSTALL_INC_DIR ${CMAKE_INSTALL_INCLUDEDIR} CACHE INTERNAL "Installation directory for headers" FORCE)
    set(INSTALL_DOC_DIR ${CMAKE_INSTALL_DOCDIR} CACHE INTERNAL "Share directory for data" FORCE)
    set(INSTALL_CMAKECONF_DIR ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
endif()

if(CMAKE_GENERATOR_TOOLSET MATCHES "v([0-9]+)_xp")
    add_definitions(-D_WIN32_WINNT=0x0501)
endif()

set(PACKAGE_NAME ${PROJECT_NAME})
string(TOUPPER ${PACKAGE_NAME} PACKAGE_UPPER_NAME)

include(util)
check_version(URI_MAJOR_VERSION URI_MINOR_VERSION URI_REV_VERSION)
set(VERSION ${URI_MAJOR_VERSION}.${URI_MINOR_VERSION}.${URI_REV_VERSION})
set(PROJECT_VERSION ${VERSION})

if(OSX_FRAMEWORK)
    set(FRAMEWORK_VERSION "${URI_MAJOR_VERSION}")
endif()
report_version(${PROJECT_NAME} ${VERSION})

configure_file(${CMAKE_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake IMMEDIATE @ONLY)

include(CheckFunctionExists)
include(CheckCCompilerFlag)
include(CheckLibraryExists)
include(CheckSymbolExists)

option(URIPARSER_BUILD_CHAR "Build code supporting data type 'char'" ON)
option(URIPARSER_BUILD_WCHAR_T "Build code supporting data type 'wchar_t'" ON)
set(URIPARSER_MSVC_RUNTIME "" CACHE STRING "Use of specific runtime library (/MT /MTd /MD /MDd) with MSVC")

if(NOT URIPARSER_BUILD_CHAR AND NOT URIPARSER_BUILD_WCHAR_T)
    message(SEND_ERROR "One or more of URIPARSER_BUILD_CHAR and URIPARSER_BUILD_WCHAR_T needs to be enabled.")
endif()
if(BUILD_TESTING AND NOT (URIPARSER_BUILD_CHAR AND URIPARSER_BUILD_WCHAR_T))
    message(SEND_ERROR "URIPARSER_BUILD_TESTS=ON requires both URIPARSER_BUILD_CHAR=ON and URIPARSER_BUILD_WCHAR_T=ON.")
endif()
if(URIPARSER_BUILD_TOOLS AND NOT URIPARSER_BUILD_CHAR)
    message(SEND_ERROR "URIPARSER_BUILD_TOOLS=ON requires URIPARSER_BUILD_CHAR=ON.")
endif()

if(MSVC AND URIPARSER_MSVC_RUNTIME)
    set(_refs
        CMAKE_CXX_FLAGS
        CMAKE_CXX_FLAGS_DEBUG
        CMAKE_CXX_FLAGS_RELEASE
        CMAKE_C_FLAGS
        CMAKE_C_FLAGS_DEBUG
        CMAKE_C_FLAGS_RELEASE
    )
    foreach(_ref ${_refs})
        string(REGEX REPLACE "/M[DT]d?" ${URIPARSER_MSVC_RUNTIME} ${_ref} "${${_ref}}")
    endforeach()
endif()

#
# Compiler checks
#
set(URIPARSER_EXTRA_COMPILE_FLAGS)

check_c_compiler_flag("-fvisibility=hidden" URIPARSER_COMPILER_SUPPORTS_VISIBILITY)
if(URIPARSER_COMPILER_SUPPORTS_VISIBILITY)
    set(URIPARSER_EXTRA_COMPILE_FLAGS "${URIPARSER_EXTRA_COMPILE_FLAGS} -fvisibility=hidden")
endif()

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${URIPARSER_EXTRA_COMPILE_FLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${URIPARSER_EXTRA_COMPILE_FLAGS}")

check_function_exists(wprintf HAVE_WPRINTF)
option(URI_SIZEDOWN "enable sizedown to save space whereever possible" OFF)
if(URI_SIZEDOWN)
    add_definitions(-DURI_SIZEDOWN)
endif()
check_function_exists(reallocarray HAVE_REALLOCARRAY)  # no luck with CheckSymbolExists
configure_file(src/UriConfig.h.in config.h)

include_directories ( ${CMAKE_CURRENT_SOURCE_DIR}/include )

set(PUBLIC_HDRS
    include/uriparser/UriBase.h
    include/uriparser/UriDefsAnsi.h
    include/uriparser/UriDefsConfig.h
    include/uriparser/UriDefsUnicode.h
    include/uriparser/Uri.h
    include/uriparser/UriIp4.h
)

set(SRCS
    src/UriCommon.c
    src/UriCommon.h
    src/UriCompare.c
    src/UriEscape.c
    src/UriFile.c
    src/UriIp4Base.c
    src/UriIp4Base.h
    src/UriIp4.c
    src/UriMemory.c
    src/UriMemory.h
    src/UriNormalizeBase.c
    src/UriNormalizeBase.h
    src/UriNormalize.c
    src/UriParseBase.c
    src/UriParseBase.h
    src/UriParse.c
    src/UriQuery.c
    src/UriRecompose.c
    src/UriResolve.c
    src/UriShorten.c
)

if(UNIX)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -O -W -Wall -ansi -pedantic")
elseif(WIN32)
    if(BUILD_SHARED_LIBS)
        set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
    # set(SRCS ${SRCS} .def)
    endif()
endif()

set(LIB_NAME ${PROJECT_NAME})
if(BUILD_SHARED_LIBS)
    add_library(${LIB_NAME} SHARED ${SRCS} ${PUBLIC_HDRS})
    set_target_properties (${LIB_NAME} PROPERTIES
        VERSION ${VERSION}
        SOVERSION ${URI_MAJOR_VERSION})
elseif(OSX_FRAMEWORK)
    set_property(SOURCE ${PUBLIC_HDRS} PROPERTY MACOSX_PACKAGE_LOCATION "Headers/uriparser")
    add_library(${LIB_NAME} SHARED ${SRCS} ${PUBLIC_HDRS})
    set_target_properties(${LIB_NAME} PROPERTIES
      FRAMEWORK TRUE
      FRAMEWORK_VERSION ${FRAMEWORK_VERSION}
      MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${VERSION}
      MACOSX_FRAMEWORK_BUNDLE_VERSION ${VERSION}
      MACOSX_FRAMEWORK_IDENTIFIER org.uriparser.uriparser
      XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
      # PUBLIC_HEADER "${PUBLIC_HDRS}"
      VERSION ${VERSION}
      SOVERSION ${URI_MAJOR_VERSION}
      )
else()
    add_library(${LIB_NAME} STATIC ${SRCS} ${PUBLIC_HDRS})
    target_compile_definitions(${LIB_NAME} PUBLIC URI_STATIC_BUILD)
endif()

target_compile_definitions(${LIB_NAME} PRIVATE URI_LIBRARY_BUILD)
if(NOT URIPARSER_BUILD_CHAR)
    target_compile_definitions(${LIB_NAME} PUBLIC URI_NO_ANSI)
endif()
if(NOT URIPARSER_BUILD_WCHAR_T)
    target_compile_definitions(${LIB_NAME} PUBLIC URI_NO_UNICODE)
endif()
if(URIPARSER_COMPILER_SUPPORTS_VISIBILITY)
    target_compile_definitions(${LIB_NAME} PRIVATE URI_VISIBILITY)
endif()

set(TARGETS ${LIB_NAME})

if(NOT SKIP_BUILD_APPS AND UNIX)
    add_executable(uriparser_exe tool/uriparse.c)
    target_link_libraries(uriparser_exe ${LIB_NAME})
    if(OSX_FRAMEWORK)
        set_target_properties(uriparser_exe PROPERTIES INSTALL_RPATH "@executable_path/../../Library/Frameworks")
    endif()
    set_target_properties(uriparser_exe PROPERTIES OUTPUT_NAME uriparser)
    set(TARGETS ${TARGETS} uriparser_exe)

    if(WIN32)
        target_link_libraries(uriparser_exe PUBLIC ws2_32)
    endif()
endif()

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    install(TARGETS ${TARGETS}
        EXPORT ${PACKAGE_UPPER_NAME}Targets
        RUNTIME DESTINATION ${INSTALL_BIN_DIR}
        LIBRARY DESTINATION ${INSTALL_LIB_DIR}
        ARCHIVE DESTINATION ${INSTALL_LIB_DIR}
        INCLUDES DESTINATION ${INSTALL_INC_DIR}
        FRAMEWORK DESTINATION ${INSTALL_LIB_DIR}
    )
endif()

if(NOT SKIP_INSTALL_HEADERS AND NOT SKIP_INSTALL_ALL)
    install(FILES ${PUBLIC_HDRS} DESTINATION "${INSTALL_INC_DIR}/uriparser")
endif()

# Testing ======================================================================
enable_testing()
if(BUILD_TESTING)
    include(FindAnyProject)
    include(CTest)
    set(API_HEADER_FILES ${PUBLIC_HDRS})
    set(LIBRARY_CODE_FILES ${SRCS})

    find_anyproject(GTest REQUIRED)

    add_executable(testrunner
        test/FourSuite.cpp
        test/MemoryManagerSuite.cpp
        test/test.cpp
        test/VersionSuite.cpp

        # These library code files have non-public symbols that the test suite
        # needs to link to, so they appear here as well:
        ${API_HEADER_FILES}
        ${LIBRARY_CODE_FILES}
    )

    target_compile_definitions(testrunner PRIVATE URI_STATIC_BUILD)

    if(MSVC)
        target_compile_definitions(testrunner PRIVATE -D_CRT_SECURE_NO_WARNINGS)
    endif()

    target_include_directories(testrunner SYSTEM PRIVATE
        ${GTEST_INCLUDE_DIRS}
    )

    target_include_directories(testrunner PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/include
        ${CMAKE_CURRENT_BINARY_DIR}  # for config.h
    )

    target_link_extlibraries(testrunner)

    add_test(
        NAME
            test
        COMMAND
            testrunner
    )

    # add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND})
endif()

# uninstall
add_custom_target(uninstall COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

# Export package ===============================================================

# Add path to includes to build-tree export
target_include_directories(${LIB_NAME} INTERFACE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  PRIVATE
      ${CMAKE_CURRENT_BINARY_DIR}
)

# Add all targets to the build-tree export set
export(TARGETS ${LIB_NAME}
    FILE ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Targets.cmake)

if(REGISTER_PACKAGE)
    # Export the package for use from the build-tree
    # (this registers the build-tree with a global CMake-registry)
    export(PACKAGE ${PACKAGE_UPPER_NAME})
endif()

# Create the <Package>Config.cmake file
configure_file(cmake/PackageConfig.cmake.in
    ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake @ONLY)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    # Install the <Package>Config.cmake
    install(FILES
      ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake
      DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)

    # Install the export set for use with the install-tree
    install(EXPORT ${PACKAGE_UPPER_NAME}Targets DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)
endif()

# Archiving ====================================================================

set(CPACK_PACKAGE_NAME "${PACKAGE_NAME}")
set(CPACK_PACKAGE_VENDOR "${PACKAGE_VENDOR}")
set(CPACK_PACKAGE_VERSION "${VERSION}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PACKAGE_NAME} Installation")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_GENERATOR "ZIP")
set(CPACK_MONOLITHIC_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)

# Get cpack zip archive name
get_cpack_filename(${VERSION} PROJECT_CPACK_FILENAME)
set(CPACK_PACKAGE_FILE_NAME ${PROJECT_CPACK_FILENAME})

include(CPack)
