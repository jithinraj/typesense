cmake_minimum_required(VERSION 2.8)
project(typesense)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -Wextra -Wno-unused-parameter -O2")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -Wall -Wextra -Wno-unused-parameter -std=c++11 -O0")
set(DEP_ROOT_DIR ${CMAKE_SOURCE_DIR}/external-${CMAKE_SYSTEM_NAME})
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/Modules/")

if(NOT EXISTS ${DEP_ROOT_DIR})
    file(MAKE_DIRECTORY ${DEP_ROOT_DIR})
endif()

if (APPLE)
    add_definitions(-DGTEST_USE_OWN_TR1_TUPLE)
    add_definitions(-D__GLIBCXX__)

    # Prefer brew installated libraries
    set(ENV{OPENSSL_ROOT_DIR} /usr/local/opt/openssl)
    set(SNAPPY_ROOT_DIR /usr/local/opt/snappy)
    set(ZLIB_ROOT /usr/local/opt/zlib)
    set(CMAKE_PREFIX_PATH /usr/local/opt/curl /usr/local/opt/icu4c)
endif (APPLE)

# Ensure that we only use the static versions of libraries to link against
# You can't statically link a shared library (or dynamically link a static library)
IF(WIN32 OR MSVC)
    SET(CMAKE_FIND_LIBRARY_SUFFIXES .lib .a)
ELSE()
    SET(CMAKE_FIND_LIBRARY_SUFFIXES .a)
ENDIF()

FIND_PACKAGE(OpenSSL 1.0.2 REQUIRED)
FIND_PACKAGE(Snappy REQUIRED)
FIND_PACKAGE(ZLIB REQUIRED)
FIND_PACKAGE(CURL REQUIRED)
FIND_PACKAGE(ICU REQUIRED)

include(cmake/For.cmake)
include(cmake/H2O.cmake)
include(cmake/RocksDB.cmake)
include(cmake/GoogleTest.cmake)
include(cmake/TestResources.cmake)
include(cmake/g3log.cmake)

FILE(GLOB SRC_FILES src/*.cpp)

include_directories(include)
include_directories(${OPENSSL_INCLUDE_DIR})
include_directories(${CURL_INCLUDE_DIR})
include_directories(${ICU_INCLUDE_DIRS})
include_directories(${DEP_ROOT_DIR}/${FOR_NAME})
include_directories(${DEP_ROOT_DIR}/${GTEST_NAME}/googletest/include)
include_directories(${DEP_ROOT_DIR}/${H2O_NAME}/include)
include_directories(${DEP_ROOT_DIR}/${ROCKSDB_NAME}/include)
include_directories(${DEP_ROOT_DIR}/${G3LOG_NAME}/src)

link_directories(${DEP_ROOT_DIR}/${GTEST_NAME}/googletest/build)
link_directories(${DEP_ROOT_DIR}/${FOR_NAME})
link_directories(${DEP_ROOT_DIR}/${H2O_NAME}/build)
link_directories(${DEP_ROOT_DIR}/${ROCKSDB_NAME})
link_directories(${DEP_ROOT_DIR}/${G3LOG_NAME}/build)

add_executable(typesense-server ${SRC_FILES} src/main/typesense_server.cpp)
add_executable(search ${SRC_FILES} src/main/main.cpp)
add_executable(benchmark ${SRC_FILES} src/main/benchmark.cpp)
add_executable(typesense_test ${SRC_FILES} test/main.cpp test/array_test.cpp test/sorted_array_test.cpp test/art_test.cpp
               test/collection_test.cpp test/collection_manager_test.cpp
               test/topster_test.cpp test/match_score_test.cpp test/store_test.cpp test/array_utils_test.cpp
               test/string_utils_test.cpp)

set(TYPESENSE_VERSION "nightly" CACHE STRING "") # will be overridden from command line during a release build

target_compile_definitions(
    typesense-server PRIVATE
    TYPESENSE_VERSION="${TYPESENSE_VERSION}"
)

target_compile_definitions(
    benchmark PRIVATE
    TYPESENSE_VERSION="${TYPESENSE_VERSION}"
)

target_compile_definitions(
    search PRIVATE
    TYPESENSE_VERSION="${TYPESENSE_VERSION}"
)

target_compile_definitions(
    typesense_test PRIVATE
    ROOT_DIR="${CMAKE_SOURCE_DIR}/"
    TYPESENSE_VERSION="${TYPESENSE_VERSION}"
)

if (APPLE)
    set(STD_LIB -static-libstdc++) # https://stackoverflow.com/a/26543140/131050 (can't statically link libgcc on Mac)

    FIND_PACKAGE(ngHTTP2 REQUIRED)
    set(CURL_LIBRARIES ldap ${NGHTTP2_LIBRARIES} ${CURL_LIBRARIES}) # ldap and nghttp2 were not being statically built
else()
    set(STD_LIB -static-libgcc -static-libstdc++)
endif()

set(ROCKSDB_LIBS rocksdb ${SNAPPY_LIBRARIES} ${ZLIB_LIBRARIES})

if(NOT APPLE)
    list(APPEND ROCKSDB_LIBS rt)
endif()

set(ICU_ALL_LIBRARIES ${ICU_I18N_LIBRARIES} ${ICU_LIBRARIES} ${ICU_DATA_LIBRARIES})

target_link_libraries(typesense-server h2o-evloop for ${ICU_ALL_LIBRARIES} ${G3LOGGER_LIBRARIES} pthread ${CURL_LIBRARIES} ${ROCKSDB_LIBS} ${OPENSSL_LIBRARIES} dl ${STD_LIB})
target_link_libraries(search for ${ICU_ALL_LIBRARIES} ${G3LOGGER_LIBRARIES} pthread h2o-evloop ${CURL_LIBRARIES} ${ROCKSDB_LIBS} ${OPENSSL_LIBRARIES} dl ${STD_LIB})
target_link_libraries(benchmark for ${ICU_ALL_LIBRARIES} ${G3LOGGER_LIBRARIES} pthread ${CURL_LIBRARIES} h2o-evloop ${ROCKSDB_LIBS} ${OPENSSL_LIBRARIES} dl ${STD_LIB})
target_link_libraries(typesense_test h2o-evloop ${ICU_ALL_LIBRARIES} ${OPENSSL_LIBRARIES} pthread for ${G3LOGGER_LIBRARIES} ${ROCKSDB_LIBS} gtest gtest_main dl ${STD_LIB})
