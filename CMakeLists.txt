cmake_minimum_required(VERSION 3.1)
include(CMakeDependentOption)

project(f3d
  VERSION "1.2.1"
  DESCRIPTION "F3D - A fast and minimalist 3D viewer"
  LANGUAGES C CXX)

string(TIMESTAMP F3D_BUILD_DATE "%Y-%m-%d %H:%M:%S" UTC)

# CMake variables
option(BUILD_TESTING "Build the tests" OFF)
cmake_dependent_option(F3D_ENABLE_LONG_TIMEOUT_TESTS "Enable long timeout tests" OFF "BUILD_TESTING" ON)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Build type" FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release")
endif()

option(F3D_INSTALL_SDK "Install F3D library SDK" OFF)

# VTK dependency
find_package(VTK 9.0 REQUIRED
  COMPONENTS
    FiltersGeneral
    FiltersGeometry
    ImagingCore
    ImagingHybrid
    InteractionStyle
    InteractionWidgets
    IOCityGML
    IOGeometry
    IOImage
    IOImport
    IOParallel
    IOPLY
    IOXML
    RenderingAnnotation
    RenderingCore
    RenderingLabel
    RenderingOpenGL2
    RenderingVolumeOpenGL2
    jsoncpp
    opengl
  OPTIONAL_COMPONENTS
    IOExodus
    RenderingRayTracing)

# Modules
option(F3D_MODULE_RAYTRACING "Raytracing module" OFF)
option(F3D_MODULE_EXODUS "ExodusII module" ON)
option(F3D_MODULE_OCCT "OpenCASCADE module (STEP and IGES files)" OFF)
option(F3D_MODULE_ASSIMP "Assimp module (FBX, OFF, DAE anf DXF files)" OFF)
option(F3D_MODULE_ALEMBIC "Alembic module (ABC files)" OFF)

set(F3D_VTK_MODULES
  VTK::CommonSystem
  VTK::FiltersGeneral
  VTK::FiltersGeometry
  VTK::ImagingCore
  VTK::ImagingHybrid
  VTK::InteractionStyle
  VTK::InteractionWidgets
  VTK::IOCityGML
  VTK::IOGeometry
  VTK::IOImage
  VTK::IOImport
  VTK::IOParallel
  VTK::IOPLY
  VTK::IOXML
  VTK::RenderingAnnotation
  VTK::RenderingCore
  VTK::RenderingLabel
  VTK::RenderingOpenGL2
  VTK::RenderingVolumeOpenGL2
  VTK::jsoncpp
  VTK::opengl
)

if(F3D_MODULE_RAYTRACING)
  if(TARGET VTK::RenderingRayTracing)
    message(STATUS "VTK RayTracing enabled")
    set(F3D_VTK_MODULES ${F3D_VTK_MODULES} VTK::RenderingRayTracing)
  else()
    message(WARNING "F3D_MODULE_RAYTRACING option is enabled but VTK::RenderingRayTracing is not found")
    set(F3D_MODULE_RAYTRACING OFF)
  endif()
endif()

if(F3D_MODULE_EXODUS)
  if(TARGET VTK::IOExodus)
    message(STATUS "Module: VTK ExodusII found")
    set(F3D_VTK_MODULES ${F3D_VTK_MODULES} VTK::IOExodus)
  else()
    message(WARNING "F3D_MODULE_EXODUS option is enabled but VTK::IOExodus is not found")
    set(F3D_MODULE_EXODUS OFF)
  endif()
endif()

set(F3D_MODULE_OCCT_XCAF 0)
if(F3D_MODULE_OCCT)
  find_package(OpenCASCADE REQUIRED)
  if(OpenCASCADE_FOUND)
    set(F3D_OCCT_VERSION "${OpenCASCADE_VERSION}")
    message(STATUS "Module: OpenCASCADE ${F3D_OCCT_VERSION} found")
    if((TARGET "TKXDESTEP") AND (TARGET "TKXDEIGES"))
      set(F3D_MODULE_OCCT_XCAF 1)
    else()
      message(WARNING "Module: OpenCASCADE: TKXDESTEP and TKXDEIGES are not found. Parts color and name are not supported.")
    endif()
    if((NOT TARGET "TKSTEP") AND (NOT TARGET "TKIGES") AND (NOT TARGET "TKMesh"))
      message(FATAL_ERROR "Module: OpenCASCADE does not contain required modules")
    endif()
  endif()
endif()

if(F3D_MODULE_ASSIMP)
  find_package(assimp 5.0 REQUIRED)
  if(assimp_FOUND)
    set(F3D_ASSIMP_VERSION "${assimp_VERSION}")
    message(STATUS "Module: assimp ${F3D_ASSIMP_VERSION} found")
    if("${F3D_ASSIMP_VERSION}" VERSION_GREATER_EQUAL "5.1.0")
      message(WARNING "Module: assimp: Animations are not working with assimp 5.1.0 and newer, use assimp 5.0.X for animation support with assimp formats.")
    endif()
    if("${F3D_ASSIMP_VERSION}" VERSION_LESS "5.1.0")
      message(WARNING "Module: assimp: Embedded texture are only working with assimp 5.1.X and newer.")
    endif()
  endif()
endif()

if(F3D_MODULE_ALEMBIC)
  find_package(Alembic 1.7 REQUIRED)
  if(Alembic_FOUND)
    set(F3D_ALEMBIC_VERSION "${Alembic_VERSION}")
    message(STATUS "Module: alembic ${Alembic_VERSION} found")
  endif()
endif()

option(F3D_GENERATE_MAN "Generate and install a man entry for f3d using help2man" OFF)
mark_as_advanced(F3D_GENERATE_MAN)

# Windows Shell Extension DLL
cmake_dependent_option(BUILD_WINDOWS_SHELL_THUMBNAILS_EXTENSION "Build the Windows Shell Extension to produce thumbnails" ON "WIN32" OFF)

# Build f3d
include(GNUInstallDirs)
add_subdirectory(src)

# testing
if(BUILD_TESTING)
  include(cmake/testing.cmake)
endif()

# installation
option(F3D_INSTALL_DEFAULT_CONFIGURATION_FILE "Install a default configuration file" OFF)
mark_as_advanced(F3D_INSTALL_DEFAULT_CONFIGURATION_FILE)
if(UNIX AND NOT APPLE)
  cmake_dependent_option(F3D_INSTALL_DEFAULT_CONFIGURATION_FILE_IN_PREFIX "Install the default configuration at the prefix root instead of system wide" OFF
    "F3D_INSTALL_DEFAULT_CONFIGURATION_FILE" OFF)
  mark_as_advanced(F3D_INSTALL_DEFAULT_CONFIGURATION_FILE_IN_PREFIX)
endif()

if(UNIX AND NOT APPLE)
  option(F3D_INSTALL_THUMBNAILER_FILES "Install thumbnailer files" OFF)
  mark_as_advanced(F3D_INSTALL_THUMBNAILER_FILES)

  option(F3D_INSTALL_MIME_TYPES_FILE "Install mime types files" OFF)
  mark_as_advanced(F3D_INSTALL_MIME_TYPES_FILE)
endif()

include(cmake/installing.cmake)

# packaging
include(cmake/packaging.cmake)
