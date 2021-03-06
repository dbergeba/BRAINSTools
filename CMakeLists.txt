cmake_minimum_required(VERSION 3.8.2 FATAL_ERROR)

#-----------------------------------------------------------------------------
# Enable C++11
#-----------------------------------------------------------------------------
set(_msg "Setting C++ standard")
message(STATUS "${_msg}")
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
message(STATUS "${_msg} - C++${CMAKE_CXX_STANDARD}")

if(POLICY CMP0048)
  cmake_policy(SET CMP0048 NEW) #project() behavior for setting VERSION variable
endif()
if(POLICY CMP0017)
  cmake_policy(SET CMP0017 NEW) #Prefer files from the CMake module directory when including from there
endif()
foreach(p
  CMP0054 # CMake 3.1 Only interpret ``if()`` arguments as variables or keywords when unquoted.
  CMP0042 # ``MACOSX_RPATH`` is enabled by default.
  CMP0025 # AppleClang vs. regular Clang
  CMP0063 # Honor visibility properties for all target types
  CMP0012 # 3.10.0
  )
  if(POLICY ${p})
    cmake_policy(SET ${p} NEW)
  endif()
endforeach()

#-----------------------------------------------------------------------------
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/CMake" ${CMAKE_MODULE_PATH})
include(ExternalProject)
include(ExternalProjectDependency)
include(ExternalProjectGenerateProjectDescription)
#-----------------------------------------------------------------------------
if(APPLE)
  ## RPATH-RPATH-RPATH
  ## https://cmake.org/Wiki/CMake_RPATH_handling
  ## Always full RPATH
  # In many cases you will want to make sure that the required libraries are
  # always found independent from LD_LIBRARY_PATH and the install location. Then
  # you can use these settings:

  # use, i.e. don't skip the full RPATH for the build tree
  set(CMAKE_SKIP_BUILD_RPATH  FALSE)

  # when building, don't use the install RPATH already
  # (but later on when installing)
  set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)

  set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")

  # add the automatically determined parts of the RPATH
  # which point to directories outside the build tree to the install RPATH
  set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)


  # the RPATH to be used when installing, but only if it's not a system directory
  list(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/lib" isSystemDir)
  if("${isSystemDir}" STREQUAL "-1")
     set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
  endif("${isSystemDir}" STREQUAL "-1")
  ## RPATH-RPATH-RPATH

  # Note: By setting CMAKE_OSX_* variables before any enable_language() or project() calls,
  #       we ensure that the bitness, and C++ standard library will be properly detected.
  include(BlockSetCMakeOSXVariables)
  mark_as_superbuild(
    VARS CMAKE_OSX_ARCHITECTURES:STRING CMAKE_OSX_SYSROOT:PATH CMAKE_OSX_DEPLOYMENT_TARGET:STRING
    ALL_PROJECTS
    )
endif()

set(LOCAL_PROJECT_NAME BRAINSTools)
set(PRIMARY_PROJECT_NAME ${LOCAL_PROJECT_NAME})

## NOTE THERE SHOULD BE NO PROJECT STATEMENT HERE!
## This file acts as a simple switch to initiate
## two completely independant CMake build environments.

#-----------------------------------------------------------------------------
# Superbuild Option - Enabled by default
#                   Phase I:  ${LOCAL_PROJECT_NAME}_SUPERBUILD is set to ON, and the
#                             supporting packages defined in "SuperBuild.cmake"
#                             are built.  The last package in "SuperBuild.cmake"
#                             to be built is a recursive call to this
#                             file with ${LOCAL_PROJECT_NAME}_SUPERBUILD explicitly
#                             set to "OFF" to initiate Phase II
#
#                   Phase II: Build the ${LOCAL_PROJECT_NAME}, referencing the support
#                             packages built in Phase I.
#-----------------------------------------------------------------------------
option(${LOCAL_PROJECT_NAME}_SUPERBUILD "Build ${LOCAL_PROJECT_NAME} and the projects it depends on via SuperBuild.cmake." ON)
mark_as_advanced(${LOCAL_PROJECT_NAME}_SUPERBUILD)

if(${LOCAL_PROJECT_NAME}_SUPERBUILD)
  project(${PRIMARY_PROJECT_NAME} C CXX)
else()
  project(${LOCAL_PROJECT_NAME} C CXX)
endif()

##-- From SimpleITK -- force try-compiles to respect language
if(POLICY CMP0067) # CMake 3.8.2
  cmake_policy(SET CMP0067 NEW)
endif()
##--

enable_language(C)
enable_language(CXX)

#-----------------------------------------------------------------------------
# Common build features for both the superbuild and the main build
#-----------------------------------------------------------------------------
include(${CMAKE_CURRENT_SOURCE_DIR}/Common.cmake)

#-----------------------------------------------------------------------------
# Sanity checks
#------------------------------------------------------------------------------
include(PreventInSourceBuilds)
include(PreventInBuildInstalls)
include(itkCheckSourceTree)

#-----------------------------------------------------------------------------
# Superbuild script
#-----------------------------------------------------------------------------
if(${LOCAL_PROJECT_NAME}_SUPERBUILD)
  set(PROJECT_NAME SuperBuild_${LOCAL_PROJECT_NAME} )
  project(${PROJECT_NAME} LANGUAGES C CXX )
  include("${CMAKE_CURRENT_SOURCE_DIR}/SuperBuild.cmake")
  return()
else()
#-----------------------------------------------------------------------------
  set(PROJECT_NAME ${LOCAL_PROJECT_NAME} )
  # Version information
  include(Version.cmake)
  set(_project_version "${${PROJECT_NAME}_VERSION_MAJOR}.${${PROJECT_NAME}_VERSION_MAJOR}.${${PROJECT_NAME}_VERSION_MAJOR}")
  project(${PROJECT_NAME} LANGUAGES  C CXX VERSION ${_project_version})

  include("${CMAKE_CURRENT_SOURCE_DIR}/${LOCAL_PROJECT_NAME}.cmake")
  return()
endif()

message(FATAL_ERROR "You should never reach this point !")

