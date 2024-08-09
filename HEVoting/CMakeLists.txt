cmake_minimum_required(VERSION 3.5)
project(HEVoting CXX)

include(CheckIncludeFileCXX)

check_include_file_cxx(any HAS_ANY)
check_include_file_cxx(string_view HAS_STRING_VIEW)
check_include_file_cxx(coroutine HAS_COROUTINE)
if (NOT "${CMAKE_CXX_STANDARD}" STREQUAL "")
    # Do nothing
elseif (HAS_ANY AND HAS_STRING_VIEW AND HAS_COROUTINE)
    set(CMAKE_CXX_STANDARD 20)
elseif (HAS_ANY AND HAS_STRING_VIEW)
    set(CMAKE_CXX_STANDARD 17)
else ()
    set(CMAKE_CXX_STANDARD 14)
endif ()

set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_executable(${PROJECT_NAME} main.cc)

#----------------------------------------------------------
######################### OPTIONS #########################
#----------------------------------------------------------

OPTION(LIBSODIUM_BUILD_SHARED_LIBRARIES "Build the library dynamically (ON/TRUE) or statically (OFF/FALSE).")
OPTION(ENABLE_BLOCKING_RANDOM           "Enable this switch only if /dev/urandom is totally broken on the target platform.")
OPTION(ENABLE_MINIMAL_BUILD             "Only compile the minimum set of functions required for the high-level API.")
OPTION(ENABLE_TESTS                     "Enable testing.")
OPTION(DISABLE_ASM                      "Disable AMD64 assembly implementations.")
OPTION(DISABLE_PIE                      "Do not produce position independent executables.")
OPTION(DISABLE_SSP                      "Do not compile with -fstack-protector.")

#----------------------------------------------------------
######################### VERSION #########################
#----------------------------------------------------------

SET(VERSION 1.0.2)
SET(SODIUM_LIBRARY_VERSION_MAJOR 7)
SET(SODIUM_LIBRARY_VERSION_MINOR 5)
SET(DLL_VERSION 6)

SET(SODIUM_LIBRARY_VERSION "1.0.2")
#                            | | |
#                     +------+ | +---+
#                     |        |     |
#                  current:revision:age
#                     |        |     |
#                     |        |     +- increment if interfaces have been added
#                     |        |        set to zero if interfaces have been removed
#                     |        |        or changed
#                     |        +- increment if source code has changed
#                     |           set to zero if current is incremented
#                     +- increment if interfaces have been added, removed or changed

CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/libsodium/src/libsodium/include/sodium/version.h.in ${CMAKE_CURRENT_SOURCE_DIR}/libsodium/src/libsodium/include/sodium/version.h)
CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/libsodium/src/libsodium/include/sodium/version.h.in ${CMAKE_CURRENT_BINARY_DIR}/libsodium/src/libsodium/include/sodium/version.h)


#----------------------------------------------------------
##################### SOURCE / HEADERS ####################
#----------------------------------------------------------

INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/libsodium/src)
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/libsodium/src/libsodium)
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/libsodium/src/libsodium/include)
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/libsodium/src/libsodium/include/sodium)
#For version.h.in to version.h
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR}/libsodium/src/libsodium/include/sodium)

file(GLOB_RECURSE libsodium_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/libsodium/src/libsodium/*.h)

file(GLOB_RECURSE libsodium_SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/libsodium/src/libsodium/*.c)

#----------------------------------------------------------
######################### LIBRARY #########################
#----------------------------------------------------------

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/build")


IF(ENABLE_BLOCKING_RANDOM)
    ADD_DEFINITIONS("-DUSE_BLOCKING_RANDOM")
ENDIF(ENABLE_BLOCKING_RANDOM)

IF(LIBSODIUM_BUILD_SHARED_LIBRARIES)
    ADD_LIBRARY(libsodium SHARED ${libsodium_HEADERS} ${libsodium_SOURCE})
ELSE(LIBSODIUM_BUILD_SHARED_LIBRARIES)
    ADD_DEFINITIONS("-DSODIUM_STATIC")
    ADD_LIBRARY(libsodium STATIC ${libsodium_HEADERS} ${libsodium_SOURCE})
ENDIF(LIBSODIUM_BUILD_SHARED_LIBRARIES)

SET_TARGET_PROPERTIES(libsodium PROPERTIES
    LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/lib
    ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/lib
    PROJECT_LABEL "libsodium"
)

########################################################################

if (CMAKE_C_COMPILER_ID STREQUAL "GNU"
    OR CMAKE_C_COMPILER_ID STREQUAL "Clang")
    set(_GCC_COMPATIBLE 1)
endif()

# static library option
if (NOT DEFINED sodium_USE_STATIC_LIBS)
    option(sodium_USE_STATIC_LIBS "enable to statically link against sodium" OFF)
endif()
if(NOT (sodium_USE_STATIC_LIBS EQUAL sodium_USE_STATIC_LIBS_LAST))
    unset(sodium_LIBRARY CACHE)
    unset(sodium_LIBRARY_DEBUG CACHE)
    unset(sodium_LIBRARY_RELEASE CACHE)
    unset(sodium_DLL_DEBUG CACHE)
    unset(sodium_DLL_RELEASE CACHE)
    set(sodium_USE_STATIC_LIBS_LAST ${sodium_USE_STATIC_LIBS} CACHE INTERNAL "internal change tracking variable")
endif()


# ##############################################################################
# If you include the drogon source code locally in your project, use this method
# to add drogon 
 add_subdirectory(drogon) 
 target_link_libraries(${PROJECT_NAME} PRIVATE drogon)

# add_subdirectory(seal) 
# target_link_libraries(${PROJECT_NAME} PRIVATE seal)
#
# and comment out the following lines
#find_package(Drogon CONFIG REQUIRED)
#target_link_libraries(${PROJECT_NAME} PRIVATE Drogon::Drogon)
find_package(SEAL 4.1.1 REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE SEAL::seal)
find_package(Boost 1.76.0 REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE Boost::boost)
#find_package(unofficial-sodium CONFIG REQUIRED)
#target_link_libraries(${PROJECT_NAME} PRIVATE unofficial-sodium::sodium)


project(libsodium VERSION 1.0.2 LANGUAGES C CXX ASM)

INCLUDE(CheckCCompilerFlag)

# ##############################################################################

if (CMAKE_CXX_STANDARD LESS 17)
    # With C++14, use boost to support any, string_view and filesystem
    message(STATUS "use c++14")
    find_package(Boost 1.61.0 REQUIRED)
    target_link_libraries(${PROJECT_NAME} PUBLIC Boost::boost)
elseif (CMAKE_CXX_STANDARD LESS 20)
    message(STATUS "use c++17")
else ()
    message(STATUS "use c++20")
endif ()

aux_source_directory(controllers CTL_SRC)
aux_source_directory(filters FILTER_SRC)
aux_source_directory(plugins PLUGIN_SRC)
aux_source_directory(models MODEL_SRC)

drogon_create_views(${PROJECT_NAME} ${CMAKE_CURRENT_SOURCE_DIR}/views
                    ${CMAKE_CURRENT_BINARY_DIR})
# use the following line to create views with namespaces.
# drogon_create_views(${PROJECT_NAME} ${CMAKE_CURRENT_SOURCE_DIR}/views
#                     ${CMAKE_CURRENT_BINARY_DIR} TRUE)
# use the following line to create views with namespace CHANGE_ME prefixed
# and path namespaces.
# drogon_create_views(${PROJECT_NAME} ${CMAKE_CURRENT_SOURCE_DIR}/views
#                     ${CMAKE_CURRENT_BINARY_DIR} TRUE CHANGE_ME)

target_include_directories(${PROJECT_NAME}
                           PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}
                                   ${CMAKE_CURRENT_SOURCE_DIR}/models)
target_sources(${PROJECT_NAME}
               PRIVATE
               ${SRC_DIR}
               ${CTL_SRC}
               ${FILTER_SRC}
               ${PLUGIN_SRC}
               ${MODEL_SRC} "util/Trie.h"  )
# ##############################################################################
# uncomment the following line for dynamically loading views 
 set_property(TARGET ${PROJECT_NAME} PROPERTY ENABLE_EXPORTS ON)

# ##############################################################################

add_subdirectory(test)
