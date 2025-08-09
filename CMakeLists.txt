cmake_minimum_required(VERSION 3.26)

project(psy_utils
        VERSION 0.1.0
        DESCRIPTION "Utility library"
        LANGUAGES CXX
)

# Enable compile_commands.json for tooling (clang-tidy, clangd, etc.)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(BUILD_SHARED_LIBS "Build libraries as shared as opposed to static" OFF)
option(UTILS_WARNINGS_AS_ERRORS "Treat warnings as errors" OFF)
option(UTILS_ENABLE_IPO "Enable Interprocedural Optimization (LTO) in Release/MinSizeRel" ON)
option(UTILS_UNITY_BUILD "Enable Unity builds for faster compilation" OFF)

add_library(${PROJECT_NAME}
        semver/PreReleasePart.hpp)
add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

set_property(TARGET ${PROJECT_NAME} PROPERTY FOLDER "${PROJECT_NAME}")

target_compile_definitions(${PROJECT_NAME}
        PUBLIC
        ${PROJECT_NAME_UPPER}_VERSION_MAJOR=${PROJECT_VERSION_MAJOR}
        ${PROJECT_NAME_UPPER}_VERSION_MINOR=${PROJECT_VERSION_MINOR}
        ${PROJECT_NAME_UPPER}_VERSION_PATCH=${PROJECT_VERSION_PATCH}
        ${PROJECT_NAME_UPPER}_VERSION_STR="${PROJECT_VERSION}"
)

# Source files
target_sources(${PROJECT_NAME} PRIVATE
        library.cpp
)

# Public header file sets (split to avoid nested BASE_DIRS in one set)
target_sources(${PROJECT_NAME}
        PUBLIC
        FILE_SET public_headers TYPE HEADERS
        BASE_DIRS
        ${CMAKE_CURRENT_SOURCE_DIR}
        FILES
        library.h
        rand/rand.hpp

        FILE_SET generated_headers TYPE HEADERS
        BASE_DIRS
        ${CMAKE_CURRENT_BINARY_DIR}
        FILES
        ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}_export.h
)

target_compile_features(${PROJECT_NAME} PUBLIC cxx_std_23)
set_target_properties(${PROJECT_NAME} PROPERTIES
        CXX_STANDARD 23
        CXX_STANDARD_REQUIRED ON
        CXX_EXTENSIONS OFF
        VERSION ${PROJECT_VERSION}
        SOVERSION ${PROJECT_VERSION_MAJOR}
)

target_include_directories(${PROJECT_NAME}
        PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
        $<INSTALL_INTERFACE:include>
)

# Warnings
if(MSVC)
    set(_warns /W4 /permissive- /Zc:__cplusplus)
    if(UTILS_WARNINGS_AS_ERRORS)
        list(APPEND _warns /WX)
    endif()
    target_compile_options(${PROJECT_NAME} PRIVATE ${_warns})
else()
    set(_warns -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion)
    if(UTILS_WARNINGS_AS_ERRORS)
        list(APPEND _warns -Werror)
    endif()
    target_compile_options(${PROJECT_NAME} PRIVATE ${_warns})
endif()

set_target_properties(${PROJECT_NAME} PROPERTIES POSITION_INDEPENDENT_CODE ON)

if(UTILS_UNITY_BUILD)
    set_target_properties(${PROJECT_NAME} PROPERTIES UNITY_BUILD ON)
endif()

if(UTILS_ENABLE_IPO)
    include(CheckIPOSupported)
    check_ipo_supported(RESULT _ipo_supported OUTPUT _ipo_msg)
    if(_ipo_supported)
        set_property(TARGET ${PROJECT_NAME} PROPERTY INTERPROCEDURAL_OPTIMIZATION_RELEASE ON)
        set_property(TARGET ${PROJECT_NAME} PROPERTY INTERPROCEDURAL_OPTIMIZATION_MINSIZEREL ON)
    else()
        message(STATUS "IPO/LTO not supported: ${_ipo_msg}")
    endif()
endif()

# Export header
include(GenerateExportHeader)

# Derive sanitized/cased project-name variants
set(PROJECT_NAME_SANITIZED "${PROJECT_NAME}")
string(REPLACE "-" "_" PROJECT_NAME_SANITIZED "${PROJECT_NAME_SANITIZED}")
string(REPLACE " " "_" PROJECT_NAME_SANITIZED "${PROJECT_NAME_SANITIZED}")

string(TOUPPER "${PROJECT_NAME_SANITIZED}" PROJECT_NAME_UPPER)
string(TOLOWER "${PROJECT_NAME_SANITIZED}" PROJECT_NAME_LOWER)

generate_export_header(${PROJECT_NAME}
        BASE_NAME ${PROJECT_NAME_UPPER}
        EXPORT_FILE_NAME ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME_LOWER}_export.h
        STATIC_DEFINE ${PROJECT_NAME_UPPER}_BUILT_AS_STATIC
)

# ---- Default symbol visibility (non-Windows) ----
# Use dllexport/dllimport on Windows via GenerateExportHeader.
# On other platforms, hide everything by default and export only what you mark.
if(NOT WIN32)
    set_target_properties(${PROJECT_NAME} PROPERTIES
            CXX_VISIBILITY_PRESET hidden
            VISIBILITY_INLINES_HIDDEN ON
    )
endif()

# ---- Windows DLL version info resource ----
# Defaults (override as cache entries or before this block if you want)
set(PROJECT_VENDOR         "Psychloor" CACHE STRING "Vendor/Company name")
set(PROJECT_DISPLAY_NAME   "${PROJECT_NAME}" CACHE STRING "Product display name")
set(PROJECT_COPYRIGHT      "© ${PROJECT_VENDOR}" CACHE STRING "Copyright notice")

# For FILEVERSION numeric fields we use PROJECT_VERSION_* from project(.. VERSION ...)
# Configure the .rc from template
if(WIN32 AND BUILD_SHARED_LIBS)
    set(VERSION_RC_IN  "${CMAKE_CURRENT_SOURCE_DIR}/cmake/version.rc.in")
    set(VERSION_RC_OUT "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}_version.rc")
    configure_file("${VERSION_RC_IN}" "${VERSION_RC_OUT}" @ONLY)

    # Add the generated resource to the target
    target_sources(${PROJECT_NAME} PRIVATE "${VERSION_RC_OUT}")
endif()

include(GNUInstallDirs)
set_target_properties(${PROJECT_NAME} PROPERTIES
        ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/$<CONFIG>/${CMAKE_INSTALL_LIBDIR}"
        LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/$<CONFIG>/${CMAKE_INSTALL_LIBDIR}"
        RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/$<CONFIG>/${CMAKE_INSTALL_BINDIR}"
)

# Install all HEADERS file sets (both will be installed)
install(
        TARGETS ${PROJECT_NAME}
        EXPORT ${PROJECT_NAME}Targets
        FILE_SET public_headers DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
        FILE_SET generated_headers DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

install(
        EXPORT ${PROJECT_NAME}Targets
        NAMESPACE ${PROJECT_NAME}::
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
)

configure_package_config_file(
        "${CMAKE_CURRENT_SOURCE_DIR}/cmake/${PROJECT_NAME}Config.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
        INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)

install(FILES
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)