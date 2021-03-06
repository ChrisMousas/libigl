cmake_minimum_required(VERSION 2.8)

project(pybind)

IF(MSVC)
### Enable parallel compilation for Visual Studio
  add_definitions(-DEIGEN_DONT_ALIGN)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP")
  SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_BINARY_DIR}/../ )
  SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_BINARY_DIR}/../ )
ENDIF(MSVC)

SET(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/../tutorial/cmake)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'MinSizeRel' as none was specified.")
  set(CMAKE_BUILD_TYPE MinSizeRel CACHE STRING "Choose the type of build." FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release"
    "MinSizeRel" "RelWithDebInfo")
endif()

find_package(OPENGL REQUIRED)

# Force a specific python version
# SET(PYTHON_LIBRARIES "D:/Python34/libs/python34.lib")
# SET(PYTHON_INCLUDE_DIR "D:/Python34/include")

# Force a specific python version
# SET(PYTHON_LIBRARIES "/usr/local/Cellar/python3/3.5.0/Frameworks/Python.framework/Versions/3.5/lib/libpython3.5m.dylib")
# SET(PYTHON_INCLUDE_DIR "/usr/local/Cellar/python3/3.5.0/Frameworks/Python.framework/Versions/3.5/include/python3.5m")

set(Python_ADDITIONAL_VERSIONS 3.4.3 3.5 3.6)
find_package(PythonLibs REQUIRED)
find_package(PythonInterp REQUIRED)

string(TOUPPER "${CMAKE_BUILD_TYPE}" U_CMAKE_BUILD_TYPE)
if (UNIX)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
  if (NOT ${U_CMAKE_BUILD_TYPE} MATCHES DEBUG)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility=hidden -flto")
  endif()
endif()

# Compile with compiler warnings turned on
if(MSVC)
  if(CMAKE_CXX_FLAGS MATCHES "/W[0-4]")
    string(REGEX REPLACE "/W[0-4]" "/W4" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
  else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4")
  endif()
else()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra")
endif()

include_directories(${PYTHON_INCLUDE_DIR} include)

## include pybing
include_directories(${PROJECT_SOURCE_DIR}/../external/pybind11/include)

## include eigen
include_directories(${PROJECT_SOURCE_DIR}/../external/nanogui/ext/eigen)

## include libigl
include_directories(${PROJECT_SOURCE_DIR}/../include)

## include glew
if(NOT APPLE)
  find_package(GLEW REQUIRED)
  include_directories( ${GLEW_INCLUDE_DIR} )
	set(SHARED_SOURCES ${SHARED_SOURCES} ${GLEW_SOURCES})
endif(NOT APPLE)


# include nanogui and dependencies
add_subdirectory("../external/nanogui/" "nanogui")
include_directories("../external/nanogui/include")
include_directories("../external/nanogui/ext/glfw/include")
include_directories("../external/nanogui/ext/nanovg/src")
list(APPEND SHARED_LIBRARIES "nanogui" "glfw" ${OPENGL_LIBRARIES})


add_library(igl SHARED
  python.cpp
  py_vector.cpp
  py_igl.cpp
  py_doc.cpp
  py_igl_viewer.cpp
  ${SHARED_SOURCES}
)

# if(NOT APPLE)
# 	find_package(GLEW REQUIRED)
# endif(NOT APPLE)

if(APPLE)
	set(CMAKE_SHARED_LINKER_FLAGS "-framework OpenGL -framework Cocoa")
endif (APPLE) #APPLE
#
# if(UNIX AND NOT APPLE)
#   set(CMAKE_CXX_LINK_EXECUTABLE "${CMAKE_CXX_LINK_EXECUTABLE} -lGL -lGLU -lrt -lX11 -lXxf86vm -lXrandr -lpthread -lXi  -lXcursor -lXinerama ")
#   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mavx")
# endif(UNIX AND NOT APPLE)


set_target_properties(igl PROPERTIES PREFIX "")
set_target_properties(igl PROPERTIES LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR})

target_link_libraries(igl ${SHARED_LIBRARIES})

if (WIN32)
  if (MSVC)
    # Enforce size-based optimization and link time code generation on MSVC (~30% smaller binaries in experiments)
    set_target_properties(igl PROPERTIES COMPILE_FLAGS "/Os /GL")
    set_target_properties(igl PROPERTIES LINK_FLAGS "/LTCG")
  endif()

  # .PYD file extension on Windows
  set_target_properties(igl PROPERTIES SUFFIX ".pyd")

  # Link against the Python shared library
  # message(FATAL_ERROR ${PYTHON_LIBRARY})
  # target_link_libraries(igl ${PYTHON_LIBRARY})
  target_link_libraries(igl ${PYTHON_LIBRARIES})


elseif (UNIX)
  # It's quite common to have multiple copies of the same Python version
  # installed on one's system. E.g.: one copy from the OS and another copy
  # that's statically linked into an application like Blender or Maya.
  # If we link our plugin library against the OS Python here and import it
  # into Blender or Maya later on, this will cause segfaults when multiple
  # conflicting Python instances are active at the same time.

  # Windows does not seem to be affected by this issue. The solution for Linux
  # and Mac OS is simple: we just don't link against the Python library. The
  # resulting shared library will have missing symbols, but that's perfectly
  # fine -- they will be resolved at import time.

  # .SO file extension on Linux/Mac OS
  set_target_properties(igl PROPERTIES SUFFIX ".so")

  # Strip unnecessary sections of the binary on Linux/Mac OS
  if(APPLE)
    set_target_properties(igl PROPERTIES MACOSX_RPATH ".")
    set_target_properties(igl PROPERTIES LINK_FLAGS "-undefined dynamic_lookup -dead_strip")
    if (NOT ${U_CMAKE_BUILD_TYPE} MATCHES DEBUG)
      add_custom_command(TARGET igl POST_BUILD COMMAND strip -u -r ${CMAKE_CURRENT_BINARY_DIR}/../igl.so)
    endif()
  else()
    if (NOT ${U_CMAKE_BUILD_TYPE} MATCHES DEBUG)
      add_custom_command(TARGET igl POST_BUILD COMMAND strip ${CMAKE_CURRENT_BINARY_DIR}/../igl.so)
    endif()
  endif()
endif()
