cmake_minimum_required(VERSION 3.15)

project(reflex VERSION 6.3.0 LANGUAGES CXX)

set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")

set(CMAKE_CXX_STANDARD 11)

include(GNUInstallDirs)

# Clang-cl uses MSVC command-line and binary interfaces despite having a
# different compiler ID.  Treat both toolchains alike when selecting targets.
set(REFLEX_MSVC_TOOLCHAIN FALSE)
if(MSVC OR CMAKE_CXX_SIMULATE_ID STREQUAL "MSVC")
  set(REFLEX_MSVC_TOOLCHAIN TRUE)
endif()

# The following sets up the simd_* variables.
include(SIMDTestAndSetup)

#
# Defining source variables
#

set(libreflex_common_sources
  lib/debug.cpp
  lib/error.cpp
  lib/input.cpp
  lib/matcher.cpp
  lib/pattern.cpp
  lib/simd.cpp
  lib/utf8.cpp
)

set(libreflex_sources
  ${libreflex_common_sources}
  lib/convert.cpp
  lib/posix.cpp
  lib/unicode.cpp
  unicode/block_scripts.cpp
  unicode/language_scripts.cpp
  unicode/letter_scripts.cpp
  unicode/letter_case.cpp
  unicode/composer.cpp
)

set(libreflexmin_sources
  ${libreflex_common_sources}
)

set(bin_sources
  src/reflex.cpp
)

set(reflex_simd_objects "")

# These noninstalled PIC objects keep AVX2 instructions out of the baseline
# sources while allowing all static and shared libraries to reuse one build.
if(REFLEX_COMPILER_HAS_AVX2)
  add_library(ReflexAvx2Objects OBJECT
    lib/matcher_avx2.cpp
    lib/simd_avx2.cpp
  )
  set_target_properties(ReflexAvx2Objects PROPERTIES POSITION_INDEPENDENT_CODE ON)
  target_include_directories(ReflexAvx2Objects PRIVATE ${PROJECT_SOURCE_DIR}/include)
  target_compile_definitions(ReflexAvx2Objects PRIVATE ${simd_definitions})
  target_compile_options(ReflexAvx2Objects PRIVATE ${simd_avx2_flags})
  list(APPEND reflex_simd_objects $<TARGET_OBJECTS:ReflexAvx2Objects>)
endif()

# AVX512BW needs a distinct compiler option and uses the AVX2 objects above as
# its runtime fallback, so it is a separate private object target as well.
if(REFLEX_COMPILER_HAS_AVX512BW)
  add_library(ReflexAvx512bwObjects OBJECT
    lib/matcher_avx512bw.cpp
    lib/simd_avx512bw.cpp
  )
  set_target_properties(ReflexAvx512bwObjects PROPERTIES POSITION_INDEPENDENT_CODE ON)
  target_include_directories(ReflexAvx512bwObjects PRIVATE ${PROJECT_SOURCE_DIR}/include)
  target_compile_definitions(ReflexAvx512bwObjects PRIVATE ${simd_definitions})
  target_compile_options(ReflexAvx512bwObjects PRIVATE ${simd_avx512bw_flags})
  list(APPEND reflex_simd_objects $<TARGET_OBJECTS:ReflexAvx512bwObjects>)
endif()

#
# Defining targets section
#

# The public headers do not declare symbols with dllexport/dllimport, so an
# MSVC-style toolchain can only produce usable static libraries at present.
if(REFLEX_MSVC_TOOLCHAIN)
  add_library(ReflexLib STATIC)
  add_library(ReflexMinLib STATIC)
  set(reflex_library_targets ReflexLib)
  set(reflexmin_library_targets ReflexMinLib)
else()
  add_library(ReflexLib SHARED)
  add_library(ReflexLibStatic STATIC)
  add_library(ReflexMinLib SHARED)
  add_library(ReflexMinLibStatic STATIC)
  set(reflex_library_targets ReflexLib ReflexLibStatic)
  set(reflexmin_library_targets ReflexMinLib ReflexMinLibStatic)
endif()

foreach(target IN LISTS reflex_library_targets)
  target_sources(${target} PRIVATE ${libreflex_sources} ${reflex_simd_objects})
endforeach()

foreach(target IN LISTS reflexmin_library_targets)
  target_sources(${target} PRIVATE ${libreflexmin_sources} ${reflex_simd_objects})
endforeach()

set(reflex_all_library_targets
  ${reflex_library_targets}
  ${reflexmin_library_targets}
)

foreach(target IN LISTS reflex_all_library_targets)
  target_include_directories(${target} PUBLIC
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  )
  target_compile_definitions(${target} PRIVATE ${simd_definitions})
  target_compile_options(${target} PRIVATE ${simd_flags})
endforeach()

add_executable(Reflex ${bin_sources})
if(TARGET ReflexLibStatic)
  target_link_libraries(Reflex PRIVATE ReflexLibStatic)
else()
  target_link_libraries(Reflex PRIVATE ReflexLib)
endif()
target_compile_definitions(Reflex PRIVATE ${simd_definitions})
target_compile_options(Reflex PRIVATE ${simd_flags})

foreach(target IN LISTS reflex_library_targets)
  set_target_properties(${target} PROPERTIES OUTPUT_NAME reflex)
endforeach()

foreach(target IN LISTS reflexmin_library_targets)
  set_target_properties(${target} PROPERTIES OUTPUT_NAME reflexmin)
endforeach()

set_target_properties(Reflex PROPERTIES OUTPUT_NAME reflex)

#
# Exporting targets section
#

install(TARGETS Reflex ${reflex_all_library_targets}
  EXPORT ReflexTargets
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

install(DIRECTORY ${PROJECT_SOURCE_DIR}/include/reflex
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  FILES_MATCHING PATTERN "*.h"
)

install(EXPORT ReflexTargets
  NAMESPACE Reflex::
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/reflex
)

#
# Packaging section (find_package support)
#

include(CMakePackageConfigHelpers)

configure_package_config_file(${PROJECT_SOURCE_DIR}/cmake/Config.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/ReflexConfig.cmake"
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/reflex
)

install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/ReflexConfig.cmake"
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/reflex
)

#
# Packaging section (pkg-config support)
#

set(prefix "${CMAKE_INSTALL_PREFIX}")
set(exec_prefix "\${prefix}")

if(IS_ABSOLUTE "${CMAKE_INSTALL_LIBDIR}")
  set(libdir "${CMAKE_INSTALL_LIBDIR}")
else()
  set(libdir "\${exec_prefix}/${CMAKE_INSTALL_LIBDIR}")
endif()

if(IS_ABSOLUTE "${CMAKE_INSTALL_INCLUDEDIR}")
  set(includedir "${CMAKE_INSTALL_INCLUDEDIR}")
else()
  set(includedir "\${prefix}/${CMAKE_INSTALL_INCLUDEDIR}")
endif()

set(REFLEX_PKGCONFIG_LIBRARY reflex)
configure_file(${PROJECT_SOURCE_DIR}/cmake/reflex.pc.in
  "${CMAKE_CURRENT_BINARY_DIR}/reflex.pc"
  @ONLY
)

set(REFLEX_PKGCONFIG_LIBRARY reflexmin)
configure_file(${PROJECT_SOURCE_DIR}/cmake/reflex.pc.in
  "${CMAKE_CURRENT_BINARY_DIR}/reflexmin.pc"
  @ONLY
)

install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/reflex.pc"
  "${CMAKE_CURRENT_BINARY_DIR}/reflexmin.pc"
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)
