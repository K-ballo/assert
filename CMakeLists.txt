# Eggs.Assert
#
# Copyright Agustin K-ballo Berge, Fusion Fenix 2026
#
# Distributed under the Boost Software License, Version 1.0. (See accompanying
# file LICENSE.txt or copy at http://www.boost.org/LICENSE_1_0.txt)

cmake_minimum_required(VERSION 3.25)
project(
    Eggs.Assert
    VERSION 1.1.0
    DESCRIPTION "A C++23 assert library with stacktrace"
    LANGUAGES C CXX
)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

include(GNUInstallDirs)

# Options (two spellings: short when top-level, prefixed when a subdirectory)
if(PROJECT_IS_TOP_LEVEL)
    set_property(GLOBAL PROPERTY USE_FOLDERS ON)
    set(CMAKE_CXX_SCAN_FOR_MODULES OFF)
    set(CMAKE_C_VISIBILITY_PRESET hidden)
    set(CMAKE_CXX_VISIBILITY_PRESET hidden)
    set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)

    set(CMAKE_MSVC_DEBUG_INFORMATION_FORMAT
        "$<$<CONFIG:Debug,RelWithDebInfo>:ProgramDatabase>"
    )

    option(BUILD_DOCS "Build the documentation" ON)
    option(BUILD_EXAMPLES "Build the examples" ON)
    option(BUILD_SHARED_LIBS "Build as a shared library" OFF)
    include(CTest)
    option(ENABLE_INSTALL "Generate install rules" ON)
else()
    option(EGGS_ASSERT_BUILD_DOCS "Build the documentation" OFF)
    option(EGGS_ASSERT_BUILD_EXAMPLES "Build the examples" OFF)
    option(
        EGGS_ASSERT_BUILD_SHARED_LIBS
        "Build as a shared library"
        ${BUILD_SHARED_LIBS}
    )
    option(EGGS_ASSERT_BUILD_TESTING "Build the test suite" OFF)
    option(EGGS_ASSERT_ENABLE_INSTALL "Generate install rules" OFF)

    # Map prefixed options back to the canonical names used below
    set(BUILD_DOCS ${EGGS_ASSERT_BUILD_DOCS})
    set(BUILD_EXAMPLES ${EGGS_ASSERT_BUILD_EXAMPLES})
    set(BUILD_SHARED_LIBS ${EGGS_ASSERT_BUILD_SHARED_LIBS})
    set(BUILD_TESTING ${EGGS_ASSERT_BUILD_TESTING})
    set(ENABLE_INSTALL ${EGGS_ASSERT_ENABLE_INSTALL})
endif()

#
# Library targets — both static and shared are always defined.
# BUILD_SHARED_LIBS selects which one Eggs::Assert aliases; the other is
# EXCLUDE_FROM_ALL (built only when a dependent target pulls it in, e.g. tests).
#
add_library(_eggs_assert_static STATIC src/assert.cpp)
add_library(_eggs_assert_shared SHARED src/assert.cpp)

foreach(_target _eggs_assert_static _eggs_assert_shared)
    set_target_properties(
        ${_target}
        PROPERTIES
            EXPORT_NAME Assert
            OUTPUT_NAME eggs_assert
            VERIFY_INTERFACE_HEADER_SETS ${BUILD_TESTING}
    )
    target_compile_features(${_target} PUBLIC cxx_std_23)
endforeach()

if(MSVC)
    set_target_properties(
        _eggs_assert_static
        PROPERTIES
            COMPILE_PDB_NAME "eggs_assert_static"
            COMPILE_PDB_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/$<CONFIG>"
    )
    set_target_properties(
        _eggs_assert_shared
        PROPERTIES
            COMPILE_PDB_NAME "eggs_assert_shared"
            COMPILE_PDB_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/$<CONFIG>"
    )
endif()

if(WIN32)
    # Both targets produce eggs_assert.lib (static archive vs DLL import library).
    # Use separate subdirectories so multi-config generators don't collide.
    set_target_properties(
        _eggs_assert_static
        PROPERTIES ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/static"
    )
    set_target_properties(
        _eggs_assert_shared
        PROPERTIES ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/shared"
    )
endif()

include(GenerateExportHeader)
generate_export_header(
    _eggs_assert_shared
    BASE_NAME EGGS_ASSERT
    EXPORT_FILE_NAME "${CMAKE_CURRENT_BINARY_DIR}/include/eggs/assert-export.h"
)
target_compile_definitions(_eggs_assert_static PUBLIC EGGS_ASSERT_STATIC_DEFINE)

foreach(_target _eggs_assert_static _eggs_assert_shared)
    target_sources(
        ${_target}
        PUBLIC
            FILE_SET HEADERS
                BASE_DIRS include "${CMAKE_CURRENT_BINARY_DIR}/include"
                FILES
                    include/eggs/assert.h
                    include/eggs/assert.hpp
                    "${CMAKE_CURRENT_BINARY_DIR}/include/eggs/assert-export.h"
    )
endforeach()

include(Cxx23Stacktrace)
foreach(_target _eggs_assert_static _eggs_assert_shared)
    target_link_libraries(${_target} PUBLIC ${EGGS_CXX23_STACKTRACE_LIB})
endforeach()

add_library(Eggs::Assert::Static ALIAS _eggs_assert_static)
add_library(Eggs::Assert::Shared ALIAS _eggs_assert_shared)
if(BUILD_SHARED_LIBS)
    add_library(Eggs::Assert ALIAS _eggs_assert_shared)
    set_target_properties(_eggs_assert_static PROPERTIES EXCLUDE_FROM_ALL ON)
else()
    add_library(Eggs::Assert ALIAS _eggs_assert_static)
    set_target_properties(_eggs_assert_shared PROPERTIES EXCLUDE_FROM_ALL ON)
endif()

add_library(_eggs_assert_intercept_static INTERFACE)
add_library(_eggs_assert_intercept_shared INTERFACE)

foreach(_target _eggs_assert_intercept_static _eggs_assert_intercept_shared)
    set_target_properties(
        ${_target}
        PROPERTIES
            EXPORT_NAME Assert::Intercept
            VERIFY_INTERFACE_HEADER_SETS ${BUILD_TESTING}
    )
    target_compile_definitions(
        ${_target}
        INTERFACE EGGS_ASSERT_INTERCEPT_ASSERT_H
    )
    target_include_directories(
        ${_target}
        SYSTEM
        BEFORE
        INTERFACE
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/eggs>
            $<INSTALL_INTERFACE:include/eggs>
    )
endforeach()

target_link_libraries(
    _eggs_assert_intercept_static
    INTERFACE _eggs_assert_static
)
target_link_libraries(
    _eggs_assert_intercept_shared
    INTERFACE _eggs_assert_shared
)

add_library(Eggs::Assert::Intercept::Static ALIAS _eggs_assert_intercept_static)
add_library(Eggs::Assert::Intercept::Shared ALIAS _eggs_assert_intercept_shared)
if(BUILD_SHARED_LIBS)
    add_library(Eggs::Assert::Intercept ALIAS _eggs_assert_intercept_shared)
else()
    add_library(Eggs::Assert::Intercept ALIAS _eggs_assert_intercept_static)
endif()

add_subdirectory(support)

#
# Documentation
#
if(BUILD_DOCS)
    add_subdirectory(docs)
endif()

#
# Example
#
if(BUILD_EXAMPLES)
    add_subdirectory(example)
endif()

#
# Tests
#
if(BUILD_TESTING)
    add_subdirectory(test)
endif()

#
# Install rules
#
if(ENABLE_INSTALL)
    include(CMakePackageConfigHelpers)

    if(BUILD_SHARED_LIBS)
        set(_eggs_assert_install_target _eggs_assert_shared)
        set(_eggs_assert_intercept_install_target _eggs_assert_intercept_shared)
    else()
        set(_eggs_assert_install_target _eggs_assert_static)
        set(_eggs_assert_intercept_install_target _eggs_assert_intercept_static)
    endif()

    install(
        TARGETS
            ${_eggs_assert_install_target}
            ${_eggs_assert_intercept_install_target}
        EXPORT EggsAssertTargets
        FILE_SET HEADERS DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )

    if(MSVC)
        if(BUILD_SHARED_LIBS)
            install(
                FILES "$<TARGET_PDB_FILE:_eggs_assert_shared>"
                DESTINATION ${CMAKE_INSTALL_BINDIR}
                CONFIGURATIONS Debug RelWithDebInfo
                OPTIONAL
            )
        else()
            install(
                FILES
                    "${CMAKE_CURRENT_BINARY_DIR}/$<CONFIG>/eggs_assert_static.pdb"
                DESTINATION ${CMAKE_INSTALL_LIBDIR}
                CONFIGURATIONS Debug RelWithDebInfo
                OPTIONAL
            )
        endif()
    endif()

    install(
        EXPORT EggsAssertTargets
        NAMESPACE Eggs::
        FILE eggs.assert-targets.cmake
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/eggs.assert
    )

    configure_package_config_file(
        cmake/eggs.assert-config.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/eggs.assert-config.cmake
        INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/eggs.assert
    )

    write_basic_package_version_file(
        ${CMAKE_CURRENT_BINARY_DIR}/eggs.assert-config-version.cmake
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
    )

    install(
        FILES
            ${CMAKE_CURRENT_BINARY_DIR}/eggs.assert-config.cmake
            ${CMAKE_CURRENT_BINARY_DIR}/eggs.assert-config-version.cmake
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/eggs.assert
    )

    set(CPACK_PACKAGE_NAME "Eggs.Assert")
    set(CPACK_PACKAGE_VENDOR "Agustin K-ballo Berge, Fusion Fenix")
    set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PROJECT_DESCRIPTION}")
    set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE.txt")
    if(WIN32)
        set(CPACK_GENERATOR ZIP)
    else()
        set(CPACK_GENERATOR TGZ)
    endif()
    include(CPack)
endif()
