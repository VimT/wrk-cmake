cmake_minimum_required(VERSION 3.17)
set(CMAKE_OSX_DEPLOYMENT_TARGET "10.9" CACHE STRING "Minimum OS X deployment version")
project(wrk)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

set(CMAKE_C_STANDARD 99)

if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release")
endif ()

set(CMAKE_C_FLAGS_RELEASE "$ENV{CFLAGS} -Wall -O3")
set(CMAKE_C_FLAGS_DEBUG "$ENV{CFLAGS} -O0 -Wall -g -ggdb")

set(WRK_LIBRARIES m ssl crypto pthread)
add_definitions(-D_REENTRANT)


if (CMAKE_SYSTEM_NAME STREQUAL "SunOS")
    add_definitions(-D_PTHREADS -D_POSIX_C_SOUCE=200112L)
    list(APPEND WRK_LIBRARIES socket)
elseif (CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set(CMAKE_EXE_LINKER_FLAGS "-pagezero_size 10000 -image_base 100000000")
elseif (CMAKE_SYSTEM_NAME STREQUAL "Linux")
    add_definitions(-D_POSIX_C_SOURCE=200112L -D_BSD_SOURCE -D_DEFAULT_SOURCE)
    list(APPEND WRK_LIBRARIES dl)
    set(CMAKE_EXE_LINKER_FLAGS "-Wl,-E")
elseif (CMAKE_SYSTEM_NAME STREQUAL "FreeBSD")
    add_definitions(-D_DECLARE_C99_LDBL_MATH)
    set(CMAKE_EXE_LINKER_FLAGS "-Wl,-E")
endif ()


find_package(LuaJIT REQUIRED)
include_directories(${LUAJIT_INCLUDE_DIR})
list(APPEND WRK_LIBRARIES ${LUAJIT_LIBRARIES})

find_package(Git)
execute_process(
        COMMAND ${GIT_EXECUTABLE} describe --tags --always --dirty
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE VERSION
        OUTPUT_STRIP_TRAILING_WHITESPACE
)

find_package(OpenSSL REQUIRED)
list(APPEND WRK_LIBRARIES OpenSSL::SSL)
include_directories(${OPENSSL_INCLUDE_DIR})

configure_file(cmake/version.c.in ${CMAKE_BINARY_DIR}/version.c @ONLY)
add_custom_command(
        OUTPUT bytecode.o
        COMMAND ${LUAJIT_BIN} -b ${CMAKE_SOURCE_DIR}/src/wrk.lua ${CMAKE_BINARY_DIR}/bytecode.o
)

SET(SOURCE
        ${CMAKE_CURRENT_SOURCE_DIR}/src/wrk.c
        ${CMAKE_CURRENT_SOURCE_DIR}/src/net.c
        ${CMAKE_CURRENT_SOURCE_DIR}/src/ssl.c
        ${CMAKE_CURRENT_SOURCE_DIR}/src/aprintf.c
        ${CMAKE_CURRENT_SOURCE_DIR}/src/stats.c
        ${CMAKE_CURRENT_SOURCE_DIR}/src/script.c
        ${CMAKE_CURRENT_SOURCE_DIR}/src/units.c
        ${CMAKE_CURRENT_SOURCE_DIR}/src/ae.c
        ${CMAKE_CURRENT_SOURCE_DIR}/src/zmalloc.c
        ${CMAKE_CURRENT_SOURCE_DIR}/src/http_parser.c
        ${CMAKE_BINARY_DIR}/bytecode.o
        ${CMAKE_BINARY_DIR}/version.c
        )

add_executable(${PROJECT_NAME} ${SOURCE})
target_link_libraries(${PROJECT_NAME} ${WRK_LIBRARIES})
