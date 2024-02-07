cmake_minimum_required(VERSION 3.1)

project(QBus CXX)

set(CMAKE_CXX_STANDARD 11)

if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif ()

set(CMAKE_INSTALL_PREFIX "${PROJECT_SOURCE_DIR}/..")
string(CONCAT CMAKE_CXX_FLAGS "-D_GLIBCXX_USE_CXX11_ABI=0 " ${CMAKE_CXX_FLAGS})
if (CMAKE_BUILD_TYPE MATCHES Debug)
    string(CONCAT CMAKE_CXX_FLAGS "-g -O0 " ${CMAKE_CXX_FLAGS})
    set(INSTALL_SUBDIR "debug")
else ()
    string(CONCAT CMAKE_CXX_FLAGS "-DNDEBUG -O2 " ${CMAKE_CXX_FLAGS})
    set(INSTALL_SUBDIR "release")
endif ()
message(STATUS "CMAKE_CXX_FLAGS: ${CMAKE_CXX_FLAGS}")

set(CMAKE_PREFIX_PATH
        ${CMAKE_PREFIX_PATH}
        ${PROJECT_SOURCE_DIR}/../thirdparts/local
)

set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

find_package(Boost REQUIRED)
message(STATUS "Boost_INCLUDE_DIRS: ${Boost_INCLUDE_DIRS}")

# find qbus::kafka's dependencies
find_path(LIBRDKAFKA_INCLUDE_DIR NAMES "librdkafka/rdkafka.h")
find_library(LIBRDKAFKA_LIB NAMES librdkafka.a)

if (NOT LIBRDKAFKA_INCLUDE_DIR OR NOT LIBRDKAFKA_LIB)
    message(FATAL_ERROR "Can't find librdkafka")
endif ()
message(STATUS "LIBRDKAFKA_INCLUDE_DIR=${LIBRDKAFKA_INCLUDE_DIR}")
message(STATUS "LIBRDKAFKA_LIB=${LIBRDKAFKA_LIB}")

find_path(LOG4CPLUS_INCLUDE_DIR NAMES "log4cplus/logger.h")
find_library(LOG4CPLUS_LIB NAMES liblog4cplus.a)
if (NOT LOG4CPLUS_INCLUDE_DIR OR NOT LOG4CPLUS_LIB)
    MESSAGE(FATAL_ERROR "Can't find log4cplus")
endif ()
add_definitions("-Wno-deprecated-declarations")

message(STATUS "LOG4CPLUS_INCLUDE_DIR=${LOG4CPLUS_INCLUDE_DIR}")
message(STATUS "LOG4CPLUS_LIB=${LOG4CPLUS_LIB}")

# find qbus::pulsar's dependencies
find_path(PULSAR_INCLUDE_PATH NAMES "pulsar/Client.h")
find_library(PULSAR_LIBRARY_PATH NAMES libpulsar.a)

# find libpulsar.a's dependencies
find_library(PROTOC_LIBRARY_PATH NAMES libprotoc.a)
find_library(PROTOBUF_LIBRARY_PATH NAMES libprotobuf.a)
find_library(PROTOBUF_LITE_LIBRARY_PATH NAMES libprotobuf-lite.a)
message(STATUS "PROTOC_LIBRARY_PATH=${PROTOC_LIBRARY_PATH}")
message(STATUS "PROTOBUF_LIBRARY_PATH=${PROTOBUF_LIBRARY_PATH}")
message(STATUS "PROTOBUF_LITE_LIBRARY_PATH=${PROTOBUF_LITE_LIBRARY_PATH}")
if (NOT PROTOC_LIBRARY_PATH OR NOT PROTOBUF_LIBRARY_PATH OR NOT PROTOBUF_LITE_LIBRARY_PATH)
    message(FATAL_ERROR "Can't find Protobuf")
endif ()
set(PROTOBUF_LIBS ${PROTOC_LIBRARY_PATH} ${PROTOBUF_LIBRARY_PATH} ${PROTOBUF_LITE_LIBRARY_PATH})

find_library(BOOST_SYSTEM_LIBRARY NAMES libboost_system.a)
find_library(BOOST_REGEX_LIBRARY NAMES libboost_regex.a)
message(STATUS "BOOST_SYSTEM_LIBRARY=${BOOST_SYSTEM_LIBRARY}")
message(STATUS "BOOST_REGEX_LIBRARY=${BOOST_REGEX_LIBRARY}")
if (NOT BOOST_SYSTEM_LIBRARY OR NOT BOOST_REGEX_LIBRARY)
    message(FATAL_ERROR "Can't find Boost.System and Boost.Regex")
endif ()
set(BOOST_LIBS ${BOOST_SYSTEM_LIBRARY} ${BOOST_REGEX_LIBRARY})

message(STATUS "Pulsar include path: ${PULSAR_INCLUDE_PATH}")
message(STATUS "Pulsar library path: ${PULSAR_LIBRARY_PATH}")
if (NOT PULSAR_INCLUDE_PATH OR NOT PULSAR_LIBRARY_PATH)
    message(FATAL_ERROR "Pulsar not found")
endif ()

set(HEADERS
        qbus_consumer.h
        qbus_producer.h
)

set(SOURCES
        kafka/qbus_config.cc
        kafka/qbus_consumer.cc
        kafka/qbus_helper.cc
        kafka/qbus_producer.cc
        kafka/qbus_producer_imp_map.cc
        kafka/qbus_rdkafka.cc
        kafka/qbus_record_msg.cc
        kafka/qbus_thread.cc
        kafka/qbus_topic_partition_set.cc
        kafka/util/logger.cc
        kafka/util/version.cc
        pulsar/log4cplus_logger.cc
        pulsar/property_tree_proxy.cc
        pulsar/pulsar_config.cc
        pulsar/qbus_consumer.cc
        pulsar/qbus_consumer_config.cc
        pulsar/qbus_php_consumer.cc
        pulsar/qbus_php_producer.cc
        pulsar/qbus_producer.cc
        pulsar/qbus_producer_config.cc
        pulsar/qbus_producer_map.cc
        pulsar/retryable_send_callback.cc
        pulsar/timer.cc
        qbus_consumer.cc
        qbus_producer.cc
        ${SOURCES}
)

set(OPENSSL_ROOT_DIR /opt/homebrew/Cellar/openssl@3/3.2.1/)
find_package(OpenSSL REQUIRED)

find_library(LZ4_LIBRARY
        NAMES lz4
        PATHS /opt/homebrew/Cellar/lz4/1.9.4/lib
        NO_DEFAULT_PATH)

find_library(ZSTD_LIBRARY
        NAMES zstd
        PATHS /opt/homebrew/Cellar/zstd/1.5.5/lib
        NO_DEFAULT_PATH)

find_library(SNAPPY_LIBRARY
        NAMES snappy
        PATHS /opt/homebrew/Cellar/snappy/1.1.10/lib/
        NO_DEFAULT_PATH)

find_library(SASL_LIBRARY
        NAMES sasl2
        PATHS /opt/homebrew/Cellar/cyrus-sasl/2.1.28_2/lib
        NO_DEFAULT_PATH)

find_library(ZLIB_LIBRAY
        NAMES z
        PATHS /opt/homebrew/Cellar/zlib/1.3/lib/
        NO_DEFAULT_PATH
)

if (NOT DEFINED LIBNAME)
    set(LIBNAME ${PROJECT_NAME})
endif ()

add_library(${LIBNAME} SHARED ${SOURCES})

set_target_properties(${LIBNAME} PROPERTIES LINK_FLAGS "-undefined dynamic_lookup")

include_directories(..
        ${EXTRA_INCLUDE_DIRS}
        ${LIBRDKAFKA_INCLUDE_DIR}
        ${LOG4CPLUS_INCLUDE_DIR}
        ${PULSAR_INCLUDE_PATH}
        ${OPENSSL_INCLUDE_DIR}
)

if (APPLE)
    target_link_libraries(${LIBNAME}
            ${EXTRA_LIBS}
            ${SASL_LIBRARY}
            ${LIBRDKAFKA_LIB}
            ${LOG4CPLUS_LIB}
            ${LZ4_LIBRARY}
            ${ZLIB_LIBRAY}
            ${ZSTD_LIBRARY}
            ${SNAPPY_LIBRARY}
            ${OPENSSL_LIBRARIES}
            # put it before PROTOBUF_LIBS and BOOST_LIBS which are its dependencies
            ${PULSAR_LIBRARY_PATH}
            ${PROTOBUF_LIBS}
            ${BOOST_LIBS}
            dl
            pthread
            #            rt
            curl
            -stdlib=libc++
    )
else ()
    target_link_libraries(${LIBNAME}
            ${EXTRA_LIBS}
            ${LIBRDKAFKA_LIB}
            ${LOG4CPLUS_LIB}
            # put it before PROTOBUF_LIBS and BOOST_LIBS which are its dependencies
            ${PULSAR_LIBRARY_PATH}
            ${PROTOBUF_LIBS}
            ${BOOST_LIBS}
            dl
            pthread
            rt
            z
            ssl
            curl
            -static-libstdc++
    )
endif ()

install(FILES ${HEADERS} DESTINATION include)
install(TARGETS ${LIBNAME} LIBRARY DESTINATION "lib/${INSTALL_SUBDIR}")
