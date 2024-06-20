cmake_minimum_required(VERSION 3.22)

project(adol-c VERSION "2.7.3")

add_definitions(-DADOLC_VERSION=2)
add_definitions(-DADOLC_SUBVERSION=7)
add_definitions(-DADOLC_PATCHLEVEL=3)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_library(adolc SHARED)
target_include_directories(adolc PUBLIC "${CMAKE_SOURCE_DIR}/ADOL-C/include/")

# Even though the files uni5_for.c, fo_rev.c, and ho_rev.c end with .c
# they are used like header files.  Together with some preprocessor trickery
# this is an old-fashioned way to do generic programming.
target_include_directories(adolc PUBLIC "${CMAKE_SOURCE_DIR}/ADOL-C/src/")

add_subdirectory(ADOL-C)

install(TARGETS adolc)
