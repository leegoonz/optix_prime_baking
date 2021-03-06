cmake_minimum_required(VERSION 2.8)


get_filename_component(PROJNAME ${CMAKE_CURRENT_SOURCE_DIR} NAME)
Project(${PROJNAME})
Message(STATUS "-------------------------------")
Message(STATUS "Processing Project ${PROJNAME}:")

SET(MODELS_DOWNLOAD_FORCE OFF CACHE BOOL "Forces the download operation of 3D models needed for this demo" )
SET(MODELS_DOWNLOAD_DISABLED ON CACHE BOOL "UNCHECK IT to download heavy 3D models needed for this demo" )
#####################################################################################
# offer the choice of having shared_sources as a sub-folder... good for packaging a sample
#
if(NOT BASE_DIRECTORY) # if not defined, it means this cmake file was called as the first entry point and not included
  # check if the external repository is outside or inside the project (as a sub-module)
  # testing the file CMakeLists_include.txt because when sub-modules are not cloned, the folders are still there...
  # we also assume here that if shared_sources is there, shared_external is, too...
  if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/shared_sources/CMakeLists_include.txt)
    Message(STATUS "found sub-folder shared_sources")
    SET(BASE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
    INCLUDE(${CMAKE_CURRENT_SOURCE_DIR}/shared_sources/CMakeLists_include.txt)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/shared_sources ${CMAKE_BINARY_DIR}/shared_sources)
    # if the resource of downloaded meshes is local, keep this one. Otherwise, use the common one
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/downloaded_resources)
      SET(DOWNLOAD_TARGET_DIR "${BASE_DIRECTORY}/downloaded_resources")
    else()
      SET(DOWNLOAD_TARGET_DIR "${BASE_DIRECTORY}/../downloaded_resources")
    endif()
  elseif(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/../../shared_sources)
    SET(BASE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/../..)
    INCLUDE(${CMAKE_CURRENT_SOURCE_DIR}/../../shared_sources/CMakeLists_include.txt)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/../../shared_sources ${CMAKE_BINARY_DIR}/shared_sources)
    SET(DOWNLOAD_TARGET_DIR "${BASE_DIRECTORY}/downloaded_resources")
  else()
    SET(BASE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/..)
    INCLUDE(${CMAKE_CURRENT_SOURCE_DIR}/../shared_sources/CMakeLists_include.txt)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/../shared_sources ${CMAKE_BINARY_DIR}/shared_sources)
    SET(DOWNLOAD_TARGET_DIR "${BASE_DIRECTORY}/downloaded_resources")
  endif()
else(NOT BASE_DIRECTORY)
    SET(DOWNLOAD_TARGET_DIR "${BASE_DIRECTORY}/downloaded_resources")
endif(NOT BASE_DIRECTORY)
_add_project_definitions(${PROJNAME})

if (SUPPORT_NVTOOLSEXT)
add_definitions(-DSUPPORT_NVTOOLSEXT)
endif(SUPPORT_NVTOOLSEXT)

#####################################################################################
# additions from packages needed for this sample
# add refs  in LIBRARIES_OPTIMIZED
# add refs  in LIBRARIES_DEBUG
# add files in PACKAGE_SOURCE_FILES
#

_add_package_Cuda () 
if( NOT CUDA_FOUND )
  message(SEND_ERROR "Baking sample requires CUDA")
endif()

_add_package_Optix ()
if ( NOT OPTIX_FOUND )
  message(SEND_ERROR "Baking sample requires OptiX")
endif()

_add_package_ZLIB()

find_package ( OpenMP )
if(OPENMP_FOUND)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
endif()

# get binary files that we don't store in github
if (NOT ${MODELS_DOWNLOAD_DISABLED})
  set(SCENE_FILES "sled_v134.bk3d.gz")
  _download_files( "${SCENE_FILES}"
                   "http://developer.download.nvidia.com/ProGraphics/nvpro-samples" 
                  "${CMAKE_CURRENT_SOURCE_DIR}/assets/" 
                  ${MODELS_DOWNLOAD_FORCE} )
endif()

# Find Eigen template lib for least squares fitting.
# If any other samples need this library in the future, then move this block
# to higher level CMakeLists.txt.

set( EIGEN3_PATH "${CMAKE_CURRENT_SOURCE_DIR}/eigen" CACHE PATH "Path to optional Eigen3 dir, for least squares filtering in baking sample" )

if ( EXISTS "${EIGEN3_PATH}/signature_of_eigen3_matrix_library" )
  set(EIGEN3_ENABLED TRUE)
else()
  set( EIGEN3_ENABLED FALSE )
endif()

if (EIGEN3_ENABLED)
  include_directories( ${EIGEN3_PATH} )
  add_definitions(-DEIGEN3_ENABLED=1)
endif()

# end Eigen


#####################################################################################
# Source files for this project
#
file(GLOB SOURCE_FILES *.cpp *.hpp *.inl *.h *.c loaders/*.cpp loaders/*.h)
file(GLOB GLSL_FILES *.glsl)
file(GLOB CUDA_FILES *.cu)

# Files included from shared_sources will disable assert in debug mode unless we define this
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -D_DEBUG")


#####################################################################################
# Executable
#
if(WIN32 AND NOT GLUT_FOUND)
  add_definitions(/wd4996) #remove printf warning
  add_definitions(/wd4244) #remove double to float conversion warning
  add_definitions(/wd4305) #remove double to float truncation warning
endif()
cuda_add_executable(${PROJNAME} ${SOURCE_FILES} ${COMMON_SOURCE_FILES} ${PACKAGE_SOURCE_FILES} ${GLSL_FILES} ${CUDA_FILES})

#####################################################################################
# common source code needed for this sample
#
source_group(common FILES 
  ${COMMON_SOURCE_FILES}
  ${PACKAGE_SOURCE_FILES}
)
source_group(shaders FILES 
  ${GLSL_FILES}
)
source_group(CUDA FILES 
  ${CUDA_FILES}
)

#####################################################################################
# Linkage
#

# Need winmm for timing code on Windows
if (WIN32) 
  LIST(APPEND PLATFORM_LIBRARIES winmm.lib)
endif()

target_link_libraries(${PROJNAME} optimized
    ${LIBRARIES_OPTIMIZED}
    ${PLATFORM_LIBRARIES}
    shared_sources
)
target_link_libraries(${PROJNAME} debug
    ${LIBRARIES_DEBUG}
    ${PLATFORM_LIBRARIES}
    shared_sources
)

#####################################################################################
# copies binaries that need to be put next to the exe files (ZLib, etc.)
#

_copy_binaries_to_target( ${PROJNAME} )

