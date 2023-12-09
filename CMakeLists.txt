cmake_minimum_required(VERSION 3.22)

# Add helper modules (Find*.cmake etc) folder to the CMake path
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})

set (LLWYTHWR_STANDALONE_PROJECT ON)
if(DEFINED PROJECT_NAME)
    set(LLWYTHWR_STANDALONE_PROJECT OFF)
endif()

## Project

project(llwythwr
        LANGUAGES C CXX
        VERSION 0.1)

set(CPM_SOURCE_CACHE "${PROJECT_SOURCE_DIR}/.cpm_cache" CACHE STRING "")

include(target_link_objects)

## Options

option(BUILD_INTERFACE_ONLY "Expose as interface library" TRUE)
option(BUILD_OBJECT_LIBS "Compile object files but do not archive or link them into a library" FALSE)

## Let BUILD_INTERFACE_ONLY override BUILD_SHARED_LIBS and BUILD_OBJECT_LIBS

if(BUILD_INTERFACE_ONLY)
    set(LIBRARY_TYPE "INTERFACE")
else()
    if (BUILD_OBJECT_LIBS)
        set(BUILD_SHARED_LIBS FALSE)
        set(LIBRARY_TYPE "OBJECT")
    endif ()
endif()

## Create the main llwythwr library target

add_library(llwythwr ${LIBRARY_TYPE})
add_library(llwythwr::llwythwr ALIAS llwythwr)

if(BUILD_INTERFACE_ONLY)
    if (WIN32)
        target_sources(llwythwr INTERFACE src/windows.c)
    elseif (APPLE)
        target_sources(llwythwr INTERFACE src/darwin.c)
    else ()
        target_sources(llwythwr INTERFACE src/linux.c)
    endif ()
    target_include_directories(llwythwr INTERFACE
            "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>")
    target_compile_definitions(llwythwr INTERFACE LLWYTHWR_INTERFACE_ONLY)
else()
    if (WIN32)
        target_sources(llwythwr PRIVATE src/windows.c)
    elseif (APPLE)
        target_sources(llwythwr PRIVATE src/darwin.c)
    else ()
        target_sources(llwythwr PRIVATE src/linux.c)
    endif ()
    set_target_properties(llwythwr PROPERTIES
            OUTPUT_NAME llwythwr
            C_VISIBILITY_PRESET hidden
            VISIBILITY_INLINES_HIDDEN ON)
    if (BUILD_SHARED_LIBS)
        set_target_properties(llwythwr PROPERTIES
                #VERSION ${llwythwr_VERSION}
                #SOVERSION ${llwythwr_VERSION_MAJOR}
                POSITION_INDEPENDENT_CODE TRUE)
    endif ()
    target_include_directories(llwythwr PUBLIC
            "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>")
    target_compile_features(llwythwr PUBLIC c_std_99)
endif()

if(UNIX AND NOT APPLE)
    find_library(LIBM m)
    if(NOT LIBM)
        message(FATAL_ERROR "libm not found and is required by llwythwr")
    endif()
    target_link_dependencies(llwythwr INTERFACE m)
    target_link_dependencies(llwythwr INTERFACE dl)
endif()

if(NOT BUILD_INTERFACE_ONLY)
    ## Generate the export header for llwythwr and attach it to the target

    include(GenerateExportHeader)
    generate_export_header(llwythwr EXPORT_FILE_NAME include/llwythwr/export.h)
    target_compile_definitions(llwythwr PUBLIC
            "$<$<NOT:$<BOOL:${BUILD_SHARED_LIBS}>>:LLWYTHWR_STATIC_DEFINE>")
    target_include_directories(llwythwr PUBLIC
            "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>")

    ## Include the install rules if the user wanted them (included by default when top-level)

    string(COMPARE EQUAL "${CMAKE_SOURCE_DIR}" "${CMAKE_CURRENT_SOURCE_DIR}" is_top_level)
    option(LLWYTHWR_INCLUDE_PACKAGING "Include packaging rules for llwythwr" "${is_top_level}")
    if (LLWYTHWR_INCLUDE_PACKAGING)
        add_subdirectory(packaging)
    endif ()
endif()

## Debug

if (CMAKE_BUILD_TYPE MATCHES Debug AND LLWYTHWR_STANDALONE_PROJECT)
    add_definitions(-DDEBUG_BUILD)
endif()

## Tests

if(CMAKE_BUILD_TYPE MATCHES Debug AND LLWYTHWR_STANDALONE_PROJECT)
    include(catch2)
    # Enable unit testing via "make test" once the code has been compiled.
    enable_testing()
    add_subdirectory(tests)
endif()