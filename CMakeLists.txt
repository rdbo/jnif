cmake_minimum_required(VERSION 3.10)

project(jnif)

option(JNIF_ZIP_SUPPORT "Enable support for zip functionality" OFF)
option(JNIF_BUILD_JNIFP "Build jnifp" OFF)
option(JNIF_BUILD_TESTS "Enable building of tests" OFF)

set(CMAKE_VERBOSE_MAKEFILE OFF)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if(NOT MSVC)
    #set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Werror")
    add_compile_options(-Wall -Wextra)
endif()

find_package(JNI)
find_package(Java QUIET)
include(UseJava)

# Resolve Z lib (if needed)
if(JNIF_ZIP_SUPPORT)
    find_package(ZLIB QUIET)
    if(NOT ZLIB_FOUND)
        include(FetchContent)
        set(ZLIB_BUILD_TESTS OFF CACHE BOOL "Disable zlib tests for FetchContent builds" FORCE)
        set(ZLIB_BUILD_EXAMPLES OFF CACHE BOOL "Disable zlib examples for FetchContent builds" FORCE)
        FetchContent_Declare(
                zlib
                URL https://www.zlib.net/zlib-1.3.1.tar.gz
                https://www.zlib.net/fossils/zlib-1.3.1.tar.gz
                https://github.com/madler/zlib/releases/download/v1.3.1/zlib-1.3.1.tar.gz
                DOWNLOAD_EXTRACT_TIMESTAMP TRUE
        )
        FetchContent_MakeAvailable(zlib)

        if(TARGET zlibstatic)
            target_include_directories(zlibstatic INTERFACE ${zlib_SOURCE_DIR} ${zlib_BINARY_DIR})
            add_library(ZLIB::ZLIB ALIAS zlibstatic)
            set(ZLIB_INCLUDE_DIRS ${zlib_SOURCE_DIR} ${zlib_BINARY_DIR})
            set(ZLIB_LIBRARIES ZLIB::ZLIB)
            set(ZLIB_FOUND ON)
        elseif(TARGET zlib)
            target_include_directories(zlib INTERFACE ${zlib_SOURCE_DIR} ${zlib_BINARY_DIR})
            add_library(ZLIB::ZLIB ALIAS zlib)
            set(ZLIB_INCLUDE_DIRS ${zlib_SOURCE_DIR} ${zlib_BINARY_DIR})
            set(ZLIB_LIBRARIES ZLIB::ZLIB)
            set(ZLIB_FOUND ON)
        endif()
    endif()

    if(NOT ZLIB_FOUND)
        message(FATAL_ERROR "ZLIB is required but was not found and could not be fetched.")
    endif()
endif()

set(JNIF_SRC
        src-libjnif/parser.cpp
        src-libjnif/base.cpp
        src-libjnif/jnif.hpp
        src-libjnif/printer.cpp
        src-libjnif/writer.cpp
        src-libjnif/stream.cpp
        src-libjnif/model.cpp
        src-libjnif/analysis.cpp)

if(JNIF_ZIP_SUPPORT)
    set(JNIF_SRC ${JNIF_SRC}
            src-libjnif/jar.cpp
            src-libjnif/zip/ioapi.c
            src-libjnif/zip/ioapi.h
            src-libjnif/zip/unzip.c
            src-libjnif/zip/unzip.h)
endif()

add_library(jnif ${JNIF_SRC})
target_include_directories(jnif INTERFACE src-libjnif)
if(JNIF_ZIP_SUPPORT)
    if(TARGET ZLIB::ZLIB)
        target_link_libraries(jnif PRIVATE ZLIB::ZLIB)
    else()
        target_include_directories(jnif PRIVATE ${ZLIB_INCLUDE_DIRS})
        target_link_libraries(jnif PRIVATE ${ZLIB_LIBRARIES})
    endif()
endif()

if(JNIF_BUILD_JNIFP)
    add_executable(jnifp
            src-jnifp/jnifp.cpp)

    target_link_libraries(jnifp jnif)
endif()

if(JNIF_BUILD_TESTS)
    add_executable(testunit
            src-testunit/testunit.cpp)

    add_executable(testjars
            src-testjars/testjars.cpp
            src-testjars/tests.cpp
            src-testjars/tests.hpp)

    add_library(testagent STATIC
            src-testagent/testagent.cpp
            src-testagent/frtlog.hpp
            src-testagent/frexception.hpp
            src-testagent/frinstr.hpp
            src-testagent/frinstrclass.cpp
            src-testagent/frinstrclient.cpp
            src-testagent/frinstrhandler.cpp
            src-testagent/frjvmti.hpp
            src-testagent/frlog.hpp
            src-testagent/frstamp.cpp
            src-testagent/frstamp.hpp
            src-testagent/frthread.cpp
            src-testagent/frthread.hpp)

    target_include_directories(testagent PRIVATE ${JNI_INCLUDE_DIRS})
    target_link_libraries(testunit jnif)
    target_link_libraries(testjars jnif)
    target_link_libraries(testagent jnif)
endif()

#add_custom_command(TARGET testagent POST_BUILD
#        COMMAND ${Java_JAVAC_EXECUTABLE} ../src-testagent/frproxy/FrInstrProxy.java)
