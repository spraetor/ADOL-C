cmake_minimum_required(VERSION 3.19)

project(adol-c
  VERSION 2.7.3
  LANGUAGES C CXX
  DESCRIPTION "A Package for Automatic Differentiation of Algorithms Written in C/C++"  # cmake >= 3.9
  HOMEPAGE_URL "https://github.com/coin-or/ADOL-C")                                     # cmake >= 3.12


add_library(adolc SHARED)
add_library(adolc::adolc ALIAS adolc)
target_compile_features(adolc PUBLIC cxx_std_17)

# Make the version of ADOL-C available as compile definitions
target_compile_definitions(adolc PRIVATE
  ADOLC_VERSION=${adol-c_VERSION_MAJOR}
  ADOLC_SUBVERSION=${adol-c_VERSION_MINOR}
  ADOLC_PATCHLEVEL=${adol-c_VERSION_PATCH})

# Set the public include directory containing headers, which will be installed
target_include_directories(adolc
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/ADOL-C/include>
    $<INSTALL_INTERFACE:include>)

# Set an include directory for the internally used library headers.
target_include_directories(adolc
  PRIVATE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/ADOL-C/src>)



# define options
# --------------

option(ENABLE_MEDIPACK "Build ADOL-C with MeDiPack (MPI) support" FALSE)
option(ENABLE_PYTHON "Build ADOL-C with python/swig support" FALSE)
option(ENABLE_ULONG "Enable 64-bit locations (only available on 64-bit systems)" FALSE)

# The boolean valued comparison operators with two adouble arguments will
# not return boolean results but the active results may be used to automatically
# switch branches in conjunction with condassign or advector (see manual).
option(ENABLE_ADVANCED_BRANCHING "Enable advanced branching operations to reduce retaping" FALSE)

# With this enabled some additional checks will be conducted when setting the
# number of directional derivatives for tapeless numbers using the SetNumDir()
# function.
option(ENABLE_TRACELESS_REFCOUNTING "Enable reference counting for tapeless numbers" FALSE)

# Only the operations involving actual dependency relationships from the
# independent variables will be recorded on the trace, this however
# requires more checks to be performed during the tracing and increases
# tracing time. Useful only if memory is a constraint and tracing is
# done fewer times than derivative computations.
option(ENABLE_ACTIVITY_TRACKING "Enable activity tracking to reduce trace size but increased tracing time" FALSE)

option(ENABLE_HARDDEBUG "Enable ADOL-C hard debug mode" FALSE)
if (ENABLE_HARDDEBUG)
  set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Choose the type of build." FORCE)
endif (ENABLE_HARDDEBUG)

# adouble constructors need to be called. With malloc or realloc that is not the case.
# With this option the adouble can do a late initialization.)
option(ENABLE_LATEINIT "Whether the adouble variables have a late initialize option for functions like malloc/realloc" FALSE)

option(DISABLE_USE_CALLOC "Disable use of calloc and use malloc instead for memory allocation" FALSE)
option(DISABLE_DOUBLE "Disable double precision arithmetic (untested)" FALSE)
option(DISABLE_STDCZERO "adouble default constructor does not initialize the value to zero" FALSE)


# handle the options
# ------------------

# Whether to build ADOL-C with adjoinable MPI (AMPI) or MeDiPack (MPI) support
target_compile_definitions(adolc PRIVATE
  $<$<BOOL:${ENABLE_MEDIPACK}>:ADOLC_MEDIPACK_SUPPORT=1>)

if(ENABLE_MEDIPACK)
  find_package(MPI REQUIRED)
  find_package(MeDiPack 1.0...<1.2 REQUIRED)                  # cmake >= 3.19
  target_link_libraries(adolc PUBLIC MeDiPack MPI::MPI_CXX)
endif()

# enable python and swig
target_compile_definitions(adolc PRIVATE
  $<$<BOOL:${ENABLE_PYTHON}>:ADOLC_python_SUPPORT=1>)

if(ENABLE_PYTHON)
  find_package(Python3 3.8 REQUIRED)
  find_package(SWIG REQUIRED COMPONENTS python)
endif()


target_compile_definitions(adolc PRIVATE
  $<$<BOOL:${ENABLE_ACTIVITY_TRACKING}>:ADOLC_TRACK_ACTIVITY=1>
  $<$<BOOL:${ENABLE_LATEINIT}>:ADOLC_ADOUBLE_LATEINIT=1>)

# Enable advanced branching operations to reduce retaping [default=no].
# The boolean valued comparison operators with two adouble arguments will
# not return boolean results but the active results may be used to automatically
# switch branches in conjunction with condassign or advector (see manual).
if(ENABLE_ADVANCED_BRANCHING)
  set(ADVBRANCH "#define ADOLC_ADVANCED_BRANCHING 1")
else()
  set(ADVBRANCH "#undef ADOLC_ADVANCED_BRANCHING")
endif()

# Enable reference counting for tapeless numbers [default=no].
# With this enabled some additional checks will be conducted when setting the
# number of directional derivatives for tapeless numbers using the SetNumDir()
# function.
if(ENABLE_TRACELESS_REFCOUNTING)
  set(ADTL_REFCNT "#define USE_ADTL_REFCOUNTING 1")
else()
  set(ADTL_REFCNT "#undef USE_ADTL_REFCOUNTING")
endif()

# Set compile flags depending on the disabled features
target_compile_definitions(adolc PRIVATE
  $<$<NOT:$<BOOL:${DISABLE_USE_CALLOC}>>:ADOLC_USE_CALLOC=1>)

# The adouble default constructor does not initialize the value to zero
# (improves performance but yields incorrect results for implicit array initializations, see manual)
target_compile_definitions(adolc PRIVATE
  $<$<NOT:$<BOOL:${DISABLE_STDCZERO}>>:ADOLC_ADOUBLE_STDCZERO=1>)

# Debug mode enabled
target_compile_definitions(adolc PRIVATE
  $<$<OR:$<CONFIG:Debug>,$<BOOL:${ENABLE_HARDDEBUG}>>:ADOLC_DEBUG=1>
  $<$<BOOL:${ENABLE_HARDDEBUG}>:ADOLC_HARDDEBUG=1>)

# define data types to use, based on the options ENABLE_ULONG and DISABLE_DOUBLE
if(ENABLE_ULONG)
  set(UINT_TYPE uint64_t)
else()
  set(UINT_TYPE uint32_t)
endif()

if (DISABLE_DOUBLE)
  set(REAL_TYPE float)
else()
  set(REAL_TYPE double)
endif()

set(SPARSE_DRIVERS "") # TODO

find_package(Boost 1.54 COMPONENTS system)
set(WITH_BOOST 0)
if(Boost_FOUND)
  target_link_libraries(adolc PUBLIC Boost::boost Boost::system)
  set(WITH_BOOST 1)

  set(USE_BOOST_POOL "") # TODO
endif()


# include subdirectories for handling of includes and source files
# ----------------------------------------------------------------

add_subdirectory(ADOL-C)


# export the targets
# ------------------

include(CMakePackageConfigHelpers)
include(GNUInstallDirs)

install(TARGETS adolc EXPORT adolcTargets)
install(EXPORT adolcTargets
  FILE adolc-targets.cmake
  NAMESPACE adolc::
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/adolc)

write_basic_package_version_file(adolc-config-version.cmake
  VERSION ${adol-c_VERSION}
  COMPATIBILITY SameMinorVersion)               # cmake >= 3.11

configure_package_config_file(adolc-config.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/adolc-config.cmake
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/adolc)

install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/adolc-config.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/adolc-config-version.cmake
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/adolc)

export(TARGETS adolc NAMESPACE adolc:: FILE adolc-targets.cmake)

# print a summary of found packages and set options
# -------------------------------------------------

include(FeatureSummary)
feature_summary(WHAT ALL)