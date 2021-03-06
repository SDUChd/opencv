cmake_minimum_required(VERSION 2.8.8 FATAL_ERROR)

include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

set(TEGRA_HAL_DIR "${CMAKE_CURRENT_SOURCE_DIR}")
set(CAROTENE_DIR "${TEGRA_HAL_DIR}/../")

if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(arm.*|ARM.*)")
  set(ARM TRUE)
elseif (CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64.*|AARCH64.*")
  set(AARCH64 TRUE)
endif()

if(ANDROID AND ARM)
  set(WITH_TGPU ON CACHE BOOL "Enable Tegra GPGPU optimization")
endif()

set(TEGRA_COMPILER_FLAGS "")

if(CMAKE_COMPILER_IS_GNUCXX)
  # Generate unwind information even for functions that can't throw/propagate exceptions.
  # This lets debuggers and such get non-broken backtraces for such functions, even without debugging symbols.
  list(APPEND TEGRA_COMPILER_FLAGS -funwind-tables)
endif()

if(CMAKE_COMPILER_IS_GNUCXX)
  if(X86 OR ARMEABI_V6 OR (MIPS AND ANDROID_COMPILER_VERSION VERSION_LESS "4.6"))
    list(APPEND TEGRA_COMPILER_FLAGS -fweb -fwrapv -frename-registers -fsched-stalled-insns-dep=100 -fsched-stalled-insns=2)
  else()
    list(APPEND TEGRA_COMPILER_FLAGS -fweb -fwrapv -frename-registers -fsched2-use-superblocks -fsched2-use-traces
                                     -fsched-stalled-insns-dep=100 -fsched-stalled-insns=2)
  endif()
  if((ANDROID_COMPILER_IS_CLANG OR NOT ANDROID_COMPILER_VERSION VERSION_LESS "4.7") AND ANDROID_NDK_RELEASE STRGREATER "r8d" )
    list(APPEND TEGRA_COMPILER_FLAGS -fgraphite -fgraphite-identity -floop-block -floop-flatten -floop-interchange
                                     -floop-strip-mine -floop-parallelize-all -ftree-loop-linear)
  endif()
endif()

if(ARM OR AARCH64)
  set(CHECK_TEGRA_HARDWARE_DEFAULT ON)
else()
  set(CHECK_TEGRA_HARDWARE_DEFAULT OFF)
endif()
set(CHECK_TEGRA_HARDWARE ${CHECK_TEGRA_HARDWARE_DEFAULT} CACHE BOOL
    "Verify Tegra platform before running optimized code")

string(REPLACE ";" " " TEGRA_COMPILER_FLAGS "${TEGRA_COMPILER_FLAGS}")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${TEGRA_COMPILER_FLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TEGRA_COMPILER_FLAGS}")

if(ANDROID_NATIVE_API_LEVEL LESS 9 AND (WITH_TGPU OR CHECK_TEGRA_HARDWARE))
  message(FATAL_ERROR "GPU support and Hardware detector is not available for API levels below 9.
Please disable Tegra GPU support and hardware detection or configure project for API level 9 or above.")
endif()

if(ARMEABI_V7A)
  if (CMAKE_COMPILER_IS_GNUCXX)
    set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-tree-vectorize" )
    set( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fno-tree-vectorize" )
  endif()
endif()

if (CHECK_TEGRA_HARDWARE)
  add_definitions(-DCHECK_TEGRA_HARDWARE)
endif()

if(WITH_TGPU)
  add_definitions(-DHAVE_TGPU)
endif()

if(WITH_LOGS)
  add_definitions(-DHAVE_LOGS)
endif()

set(CAROTENE_NS "carotene_o4t" CACHE STRING "" FORCE)

function(compile_carotene)
  if(ENABLE_NEON)
    set(WITH_NEON ON)
  endif()

  add_subdirectory("${CAROTENE_DIR}" "${CMAKE_CURRENT_BINARY_DIR}/carotene")

  if(ARM OR AARCH64)
    if(CMAKE_BUILD_TYPE)
      set(CMAKE_TRY_COMPILE_CONFIGURATION ${CMAKE_BUILD_TYPE})
    endif()
    check_cxx_compiler_flag("-mfpu=neon" CXX_HAS_MFPU_NEON)
    check_c_compiler_flag("-mfpu=neon" C_HAS_MFPU_NEON)
    if(${CXX_HAS_MFPU_NEON} AND ${C_HAS_MFPU_NEON})
      get_target_property(old_flags "carotene_objs" COMPILE_FLAGS)
      if(old_flags)
        set_target_properties("carotene_objs" PROPERTIES COMPILE_FLAGS "${old_flags} -mfpu=neon")
      else()
        set_target_properties("carotene_objs" PROPERTIES COMPILE_FLAGS "-mfpu=neon")
      endif()
    endif()
  endif()
endfunction()

compile_carotene()

include_directories("${CAROTENE_DIR}/include")

get_target_property(carotene_defs carotene_objs INTERFACE_COMPILE_DEFINITIONS)
set_property(DIRECTORY APPEND PROPERTY COMPILE_DEFINITIONS ${carotene_defs})

  if (CMAKE_COMPILER_IS_GNUCXX)
    # allow more inlines - these parameters improve performance for:
    #   matchTemplate about 5-10%
    #   goodFeaturesToTrack 10-20%
    #   cornerHarris 30% for some cases
    set_source_files_properties(impl.cpp $<TARGET_OBJECTS:carotene_objs> COMPILE_FLAGS "--param ipcp-unit-growth=100000 --param inline-unit-growth=100000 --param large-stack-frame-growth=5000")
#    set_source_files_properties(impl.cpp $<TARGET_OBJECTS:carotene_objs> COMPILE_FLAGS "--param ipcp-unit-growth=100000 --param inline-unit-growth=100000 --param large-stack-frame-growth=5000")
  endif()

add_library(tegra_hal STATIC $<TARGET_OBJECTS:carotene_objs>)
set_target_properties(tegra_hal PROPERTIES POSITION_INDEPENDENT_CODE TRUE)
set_target_properties(tegra_hal PROPERTIES ARCHIVE_OUTPUT_DIRECTORY ${3P_LIBRARY_OUTPUT_PATH})
set(OPENCV_SRC_DIR "${CMAKE_SOURCE_DIR}")
if(NOT BUILD_SHARED_LIBS)
  ocv_install_target(tegra_hal EXPORT OpenCVModules ARCHIVE DESTINATION ${OPENCV_3P_LIB_INSTALL_PATH} COMPONENT dev)
endif()
target_include_directories(tegra_hal PRIVATE ${CMAKE_CURRENT_SOURCE_DIR} ${OPENCV_SRC_DIR}/modules/core/include)

set(CAROTENE_HAL_VERSION "0.0.1" PARENT_SCOPE)
set(CAROTENE_HAL_LIBRARIES "tegra_hal" PARENT_SCOPE)
set(CAROTENE_HAL_HEADERS "carotene/tegra_hal.hpp" PARENT_SCOPE)
set(CAROTENE_HAL_INCLUDE_DIRS "${CMAKE_BINARY_DIR}" PARENT_SCOPE)

configure_file("tegra_hal.hpp" "${CMAKE_BINARY_DIR}/carotene/tegra_hal.hpp" COPYONLY)
configure_file("${CAROTENE_DIR}/include/carotene/definitions.hpp" "${CMAKE_BINARY_DIR}/carotene/definitions.hpp" COPYONLY)
configure_file("${CAROTENE_DIR}/include/carotene/functions.hpp" "${CMAKE_BINARY_DIR}/carotene/functions.hpp" COPYONLY)
configure_file("${CAROTENE_DIR}/include/carotene/types.hpp" "${CMAKE_BINARY_DIR}/carotene/types.hpp" COPYONLY)
