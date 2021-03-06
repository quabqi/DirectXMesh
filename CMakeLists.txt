﻿# DirectXMesh geometry Library
#
# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT License.
#
# http://go.microsoft.com/fwlink/?LinkID=324981

cmake_minimum_required (VERSION 3.11)

project (DirectXMesh LANGUAGES CXX)

# Includes the support for DX 12 input layouts
option(BUILD_DX12 "Build with DirectX12 Runtime support" ON)

option(ENABLE_CODE_ANALYSIS "Use Static Code Analysis on build" OFF)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/bin/CMake")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/bin/CMake")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/bin/CMake")

set(LIBRARY_SOURCES
    DirectXMesh/DirectXMesh.h
    DirectXMesh/DirectXMeshP.h
    DirectXMesh/scoped.h
    DirectXMesh/DirectXMeshAdjacency.cpp
    DirectXMesh/DirectXMeshClean.cpp
    DirectXMesh/DirectXMeshGSAdjacency.cpp
    DirectXMesh/DirectXMeshletGenerator.cpp
    DirectXMesh/DirectXMeshNormals.cpp
    DirectXMesh/DirectXMeshOptimize.cpp
    DirectXMesh/DirectXMeshOptimizeLRU.cpp
    DirectXMesh/DirectXMeshOptimizeTVC.cpp
    DirectXMesh/DirectXMeshRemap.cpp
    DirectXMesh/DirectXMeshTangentFrame.cpp
    DirectXMesh/DirectXMeshUtil.cpp
    DirectXMesh/DirectXMeshValidate.cpp
    DirectXMesh/DirectXMeshVBReader.cpp
    DirectXMesh/DirectXMeshVBWriter.cpp
    DirectXMesh/DirectXMeshWeldVertices.cpp)

add_library(${PROJECT_NAME} STATIC ${LIBRARY_SOURCES})

source_group(${PROJECT_NAME} REGULAR_EXPRESSION DirectXMesh/*.*)

target_include_directories(${PROJECT_NAME} PUBLIC DirectXMesh)

if (${CMAKE_VERSION} VERSION_GREATER_EQUAL "3.16")
    target_precompile_headers(${PROJECT_NAME} PRIVATE DirectXMesh/DirectXMeshP.h)
endif()

if(MSVC)
    # Use max Warning Level 
    string(REPLACE "/W3 " "/Wall " CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})
    string(REPLACE "/W3 " "/Wall " CMAKE_CXX_FLAGS_DEBUG ${CMAKE_CXX_FLAGS_DEBUG})
    string(REPLACE "/W3 " "/Wall " CMAKE_CXX_FLAGS_RELEASE ${CMAKE_CXX_FLAGS_RELEASE})

    # Not using typeid or dynamic_cast, so disable RTTI to save binary size
    string(REPLACE "/GR " "/GR- " CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})
    string(REPLACE "/GR " "/GR- " CMAKE_CXX_FLAGS_DEBUG ${CMAKE_CXX_FLAGS_DEBUG})
    string(REPLACE "/GR " "/GR- " CMAKE_CXX_FLAGS_RELEASE ${CMAKE_CXX_FLAGS_RELEASE})
endif()

add_executable(meshconvert
    Meshconvert/Meshconvert.cpp
    Meshconvert/MeshOBJ.cpp
    Meshconvert/Mesh.h
    Meshconvert/Mesh.cpp
    Meshconvert/SDKMesh.h)
target_include_directories(meshconvert PUBLIC MeshConvert Utilities)
target_link_libraries(meshconvert ${PROJECT_NAME})
source_group(meshconvert REGULAR_EXPRESSION meshconvert/*.*)

if(MSVC)
    target_compile_options(${PROJECT_NAME} PRIVATE /fp:fast)
    target_compile_options(meshconvert PRIVATE /fp:fast)

    if(${CMAKE_SIZEOF_VOID_P} EQUAL "4")
        target_compile_options(${PROJECT_NAME} PRIVATE /arch:SSE2)
        target_compile_options(meshconvert PRIVATE /arch:SSE2)
    endif()
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set(WarningsLib "-Wpedantic" "-Wextra")
    target_compile_options(${PROJECT_NAME} PRIVATE ${WarningsLib})

    set(WarningsEXE ${WarningsLib} "-Wno-c++98-compat" "-Wno-c++98-compat-pedantic" "-Wno-switch" "-Wno-switch-enum" "-Wno-double-promotion" "-Wno-exit-time-destructors" "-Wno-missing-prototypes")
    target_compile_options(meshconvert PRIVATE ${WarningsEXE})
endif()
if (CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    target_compile_options(${PROJECT_NAME} PRIVATE /permissive- /JMC- /Zc:__cplusplus)
    target_compile_options(meshconvert PRIVATE /permissive- /JMC- /Zc:__cplusplus)

   if(ENABLE_CODE_ANALYSIS)
      target_compile_options(${PROJECT_NAME} PRIVATE /analyze)
      target_compile_options(meshconvert PRIVATE /analyze)
   endif()

    if (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 19.26)
        target_compile_options(${PROJECT_NAME} PRIVATE /Zc:preprocessor /wd5105)
        target_compile_options(meshconvert PRIVATE /Zc:preprocessor /wd5105)
    endif()

    set(WarningsEXE "/wd4061" "/wd4365" "/wd4668" "/wd4710" "/wd4820" "/wd5039" "/wd5045" "/wd5219")
    target_compile_options(meshconvert PRIVATE ${WarningsEXE})
endif()

if(WIN32)
    # Windows 10 is used here to build the DirectX 12 code paths as well as 11
    target_compile_definitions(${PROJECT_NAME} PRIVATE _UNICODE UNICODE)
    target_compile_definitions(meshconvert PRIVATE _UNICODE UNICODE _WIN32_WINNT=0x0601)

    if(BUILD_DX12)
        target_compile_definitions(${PROJECT_NAME} PRIVATE _WIN32_WINNT=0x0A00)
    else()
        target_compile_definitions(${PROJECT_NAME} PRIVATE _WIN32_WINNT=0x0601)
    endif()
endif()
