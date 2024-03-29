# Copyright (C) 2011 The Libphonenumber Authors
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Author: Philippe Liard

cmake_minimum_required (VERSION 2.8)

project (libphonenumber)
set (libphonenumber_VERSION_MAJOR 5)
set (libphonenumber_VERSION_MINOR 2)

# Helper functions dealing with finding libraries and programs this library
# depends on.

include (../tools/cpp/gtest.cmake)

function (print_error DESCRIPTION FILE)
  message (FATAL_ERROR
    "Can't find ${DESCRIPTION}: can't locate ${FILE}. Please read the README.")
endfunction ()

# Find a library. If it has not been found, stop CMake with a fatal error
# message.
function (find_required_library NAME HEADER LIBRARY DESCRIPTION)
  # Check the header.
  find_path (${NAME}_INCLUDE_DIR ${HEADER})
  set (INCLUDE_DIR ${${NAME}_INCLUDE_DIR})

  if (${INCLUDE_DIR} STREQUAL "${INCLUDE_DIR}-NOTFOUND")
    print_error (${DESCRIPTION} ${HEADER})
  endif ()
  include_directories (${INCLUDE_DIR})
  # Check the binary.
  find_library (${NAME}_LIB ${LIBRARY})
  set (LIB ${NAME}_LIB)

  if (${LIB} STREQUAL "${LIB}-NOTFOUND")
    print_error (${DESCRIPTION} ${LIBRARY})
  endif ()
endfunction (find_required_library)

# Check the library version (if pkg-config available).
find_package (PkgConfig)
function (check_library_version VARNAME LIBRARY_WITH_VERSION)
  if (PKG_CONFIG_FOUND)
    pkg_check_modules (${VARNAME} ${LIBRARY_WITH_VERSION})
  endif ()
endfunction ()

# Find a program. If it has not been found, stop CMake with a fatal error
# message.
function (find_required_program NAME FILENAME DESCRIPTION)
  find_program (${NAME}_BIN NAMES ${FILENAME})

  if (${NAME}_BIN STREQUAL "${${NAME}_BIN}-NOTFOUND")
    print_error (${DESCRIPTION} ${FILENAME})
  endif ()
endfunction (find_required_program)

# Options that can be passed to CMake using 'cmake -DKEY=VALUE'.
option ("USE_ICU_REGEXP" "Use ICU regexp engine" "ON")
option ("USE_LITE_METADATA" "Use lite metadata" "OFF")
option ("USE_RE2" "Use RE2" "OFF")
option ("USE_STD_MAP" "Force the use of std::map" "OFF")

# Find all the required libraries and programs.
if (WIN32)
  set (Boost_USE_STATIC_LIBS ON)
endif ()
find_package (Boost 1.40.0 COMPONENTS date_time thread)
if (NOT Boost_FOUND)
  print_error ("Boost Datetime/Thread" "Boost")
endif ()
include_directories (${Boost_INCLUDE_DIRS})

find_or_build_gtest ()

if (${USE_RE2} STREQUAL "ON")
  find_required_library (RE2 re2/re2.h re2 "Google RE2")
endif ()

find_required_library (PROTOBUF google/protobuf/message_lite.h protobuf
                       "Google Protocol Buffers")
check_library_version (PC_PROTOBUF protobuf>=2.4)

find_required_library (ICU_UC unicode/uchar.h icuuc "ICU")
check_library_version (PC_ICU_UC icu-uc>=4.4)

set (ICU_INCLUDE_DIR ${ICU_UC_INCLUDE_DIR})
set (ICU_LIB ${ICU_UC_LIB})
# If ICU regexp engine is used, use icui18n as well.
if (${USE_ICU_REGEXP} STREQUAL "ON")
  find_required_library (ICU_I18N unicode/regex.h icui18n "ICU")
  check_library_version (PC_ICU_I18N icu-i18n>=4.4)
  list (APPEND ICU_INCLUDE_DIR ${ICU_I18N_INCLUDE_DIR})
  list (APPEND ICU_LIB ${ICU_I18N_LIB})
endif ()

find_required_program (PROTOC protoc
                       "Google Protocol Buffers compiler (protoc)")

find_required_program (JAVA java
                       "Java Runtime Environment")

if (APPLE)
  FIND_LIBRARY (COREFOUNDATION_LIB CoreFoundation)
  FIND_LIBRARY (FOUNDATION_LIB Foundation)
endif ()

if (${USE_STD_MAP} STREQUAL "OFF")
  INCLUDE (CheckIncludeFileCXX)
  CHECK_INCLUDE_FILE_CXX ("tr1/unordered_map" HAVE_CXX_TR1_UNORDERED_MAP)
  if (HAVE_CXX_TR1_UNORDERED_MAP)
    add_definitions ("-DUSE_TR1_UNORDERED_MAP")
  endif ()
endif ()

# Add protoc (Protocol Buffers compiler) target.
set (RESOURCES_DIR "${CMAKE_SOURCE_DIR}/../resources")

set (
  PROTOBUF_SOURCES "${RESOURCES_DIR}/phonemetadata.proto"
                   "${RESOURCES_DIR}/phonenumber.proto"
)

set (
  PROTOBUF_OUTPUT "${CMAKE_SOURCE_DIR}/src/phonenumbers/phonemetadata.pb.cc"
                  "${CMAKE_SOURCE_DIR}/src/phonenumbers/phonemetadata.pb.h"
                  "${CMAKE_SOURCE_DIR}/src/phonenumbers/phonenumber.pb.cc"
                  "${CMAKE_SOURCE_DIR}/src/phonenumbers/phonenumber.pb.h"
)

add_custom_command (
  COMMAND ${PROTOC_BIN} --cpp_out=${CMAKE_SOURCE_DIR}/src/phonenumbers/
    --proto_path=${RESOURCES_DIR} ${PROTOBUF_SOURCES}

  OUTPUT ${PROTOBUF_OUTPUT}
  DEPENDS ${PROTOBUF_SOURCES}
)

add_custom_target (
  generate-sources

  DEPENDS ${PROTOBUF_OUTPUT}
  COMMENT "Generating Protocol Buffers code"
)

# Geocoding data cpp file generation
set (TOOLS_DIR "${CMAKE_CURRENT_BINARY_DIR}/tools")
add_subdirectory("${CMAKE_SOURCE_DIR}/../tools/cpp" "${TOOLS_DIR}")

set (GEOCODING_DIR "${RESOURCES_DIR}/geocoding")
file (GLOB_RECURSE GEOCODING_SOURCES "${GEOCODING_DIR}/*.txt")

set (GEOCODING_DATA_OUTPUT
  "${CMAKE_SOURCE_DIR}/src/phonenumbers/geocoding/geocoding_data.cc"
)

add_custom_command (
  COMMAND generate_geocoding_data "${GEOCODING_DIR}" "${GEOCODING_DATA_OUTPUT}"

  OUTPUT ${GEOCODING_DATA_OUTPUT}
  DEPENDS ${GEOCODING_SOURCES}
          generate_geocoding_data
  COMMENT "Generating geocoding data code"
)

set (
  SOURCES
  "src/base/string_piece.cc"
  "src/phonenumbers/asyoutypeformatter.cc"
  "src/phonenumbers/default_logger.cc"
  "src/phonenumbers/geocoding/area_code_map.cc"
  "src/phonenumbers/geocoding/default_map_storage.cc"
  "src/phonenumbers/geocoding/geocoding_data.cc"
  "src/phonenumbers/geocoding/mapping_file_provider.cc"
  "src/phonenumbers/logger.cc"
  "src/phonenumbers/phonemetadata.pb.cc" # Generated by Protocol Buffers.
  "src/phonenumbers/phonenumber.cc"
  "src/phonenumbers/phonenumber.pb.cc"   # Generated by Protocol Buffers.
  "src/phonenumbers/phonenumberutil.cc"
  "src/phonenumbers/regexp_cache.cc"
  "src/phonenumbers/shortnumberutil.cc"
  "src/phonenumbers/string_byte_sink.cc"
  "src/phonenumbers/stringutil.cc"
  "src/phonenumbers/unicodestring.cc"
  "src/phonenumbers/utf/rune.c"
  "src/phonenumbers/utf/unicodetext.cc"
  "src/phonenumbers/utf/unilib.cc"
)

# Add regexp engine-dependent sources. ICU is used by default.
if (${USE_RE2} STREQUAL "ON")
  # Add a flag to select the right regexp factory implementation used by
  # regexp_factory.h and regexp_adapter_test.cc.
  # When both ICU regexp and RE2 are defined, the regexp engine adapter defaults
  # to RE2 unless the ICU implementation is instantiated explictly obviously.
  add_definitions (-DUSE_RE2)
  list (APPEND SOURCES "src/phonenumbers/regexp_adapter_re2.cc")
endif ()

if (${USE_ICU_REGEXP} STREQUAL "ON")
  add_definitions (-DUSE_ICU_REGEXP)
  list (APPEND SOURCES "src/phonenumbers/regexp_adapter_icu.cc")
  # The phone number matcher needs ICU.
  list (APPEND SOURCES "src/phonenumbers/alternate_format.cc")
  list (APPEND SOURCES "src/phonenumbers/phonenumbermatch.cc")
  list (APPEND SOURCES "src/phonenumbers/phonenumbermatcher.cc")
endif ()

# Library sources excluding the metadata files, since special metadata is used
# for unit-testing.
set (TESTING_LIBRARY_SOURCES ${SOURCES})

# Add metadata code generation targets.

# This function is invoked to create metadata, test metadata and lite metadata
# code generation targets.
function (add_metadata_gen_target TARGET_NAME
                                  XML_FILE
                                  METADATA_TYPE
                                  METADATA_HEADER)
  set (METADATA_SOURCE_DIR "${CMAKE_SOURCE_DIR}/src/phonenumbers")
  set (GEN_OUTPUT "${METADATA_SOURCE_DIR}/${METADATA_TYPE}.cc"
                  "${METADATA_SOURCE_DIR}/${METADATA_HEADER}.h")
  set (JAR_PATH "${CMAKE_SOURCE_DIR}/../tools/java/cpp-build/target")
  set (JAR_PATH "${JAR_PATH}/cpp-build-1.0-SNAPSHOT-jar-with-dependencies.jar")

  add_custom_command (
    COMMAND ${JAVA_BIN} -jar
      ${JAR_PATH} BuildMetadataCppFromXml ${XML_FILE}
      ${CMAKE_SOURCE_DIR}/src/phonenumbers ${METADATA_TYPE}

    OUTPUT ${GEN_OUTPUT}
    DEPENDS ${XML_FILE}
  )
  add_custom_target (
    ${TARGET_NAME}
    DEPENDS ${GEN_OUTPUT}
    COMMENT "Generating Metadata code"
  )
endfunction (add_metadata_gen_target)

if (${USE_LITE_METADATA} STREQUAL "ON")
  # Add the lite metadata generation target.
  set (METADATA_TARGET "generate-lite-metadata")
  add_metadata_gen_target (
    ${METADATA_TARGET}
    "${RESOURCES_DIR}/PhoneNumberMetaData.xml"
    "lite_metadata"
    "metadata"
  )
  list (APPEND SOURCES "src/phonenumbers/lite_metadata.cc")
else ()
  # Add the metadata generation target.
  set (METADATA_TARGET "generate-metadata")
  add_metadata_gen_target (
    ${METADATA_TARGET}
    "${RESOURCES_DIR}/PhoneNumberMetaData.xml"
    "metadata"
    "metadata"
  )
  list (APPEND SOURCES "src/phonenumbers/metadata.cc")
endif ()

# Add the test metadata generation target.
set (TEST_METADATA_TARGET "generate-test-metadata")
add_metadata_gen_target (
  ${TEST_METADATA_TARGET}
  "${RESOURCES_DIR}/PhoneNumberMetaDataForTesting.xml"
  "test_metadata"
  "metadata"
)
list (APPEND TESTING_LIBRARY_SOURCES "src/phonenumbers/test_metadata.cc")

if (${USE_ICU_REGEXP} STREQUAL "ON")
  # Add alternate format metadata generation for the phone number matcher
  set (ALT_FORMAT_METADATA_TARGET "generate-alt-format-metadata")
  add_metadata_gen_target (
    ${ALT_FORMAT_METADATA_TARGET}
    "${RESOURCES_DIR}/PhoneNumberAlternateFormats.xml"
    "alternate_format"
    "alternate_format"
  )
endif ()

if (NOT WIN32)
  add_definitions ("-Wall -Werror")
endif ()

include_directories ("src")

# Build a static library (without -fPIC).
add_library (phonenumber STATIC ${SOURCES})
add_dependencies (phonenumber generate-sources ${METADATA_TARGET})
if (${USE_ICU_REGEXP} STREQUAL "ON")
  add_dependencies (phonenumber ${ALT_FORMAT_METADATA_TARGET})
endif ()

# Build a shared library (with -fPIC).
set (BUILD_SHARED_LIB true)

if (${USE_RE2} STREQUAL "ON")
  # RE2 is not always available as a shared library (e.g: package provided by
  # Ubuntu) therefore disable the shared library build in this case.
  if (${RE2_LIB} MATCHES ".*\\.a")
    message (WARNING
      "RE2 not available as a shared library, shared library build disabled")
    set (BUILD_SHARED_LIB false)
  endif ()
endif ()

if (BUILD_SHARED_LIB)
  add_library (phonenumber-shared SHARED ${SOURCES})
  add_dependencies (phonenumber-shared generate-sources ${METADATA_TARGET})
  if (${USE_ICU_REGEXP} STREQUAL "ON")
    add_dependencies (phonenumber ${ALT_FORMAT_METADATA_TARGET})
  endif ()
  set_target_properties (phonenumber-shared
    PROPERTIES
      OUTPUT_NAME "phonenumber"
      PREFIX "lib"
      SOVERSION ${libphonenumber_VERSION_MAJOR}
      VERSION ${libphonenumber_VERSION_MAJOR}.${libphonenumber_VERSION_MINOR})
endif ()

set (LIBRARY_DEPS ${PROTOBUF_LIB} ${ICU_LIB} ${Boost_LIBRARIES})

if (${USE_RE2} STREQUAL "ON")
  list (APPEND LIBRARY_DEPS ${RE2_LIB})
endif ()

if (APPLE)
  list (APPEND LIBRARY_DEPS ${COREFOUNDATION_LIB} ${FOUNDATION_LIB})
endif ()

target_link_libraries (phonenumber ${LIBRARY_DEPS})

if (BUILD_SHARED_LIB)
  target_link_libraries (phonenumber-shared ${LIBRARY_DEPS})
endif ()

# Build a specific library for testing purposes.
add_library (phonenumber_testing STATIC ${TESTING_LIBRARY_SOURCES})
target_link_libraries (phonenumber_testing ${LIBRARY_DEPS})
add_dependencies (phonenumber_testing generate-sources ${TEST_METADATA_TARGET})

# Test geocoding data cpp files generation
set (GEOCODING_TEST_DIR "${RESOURCES_DIR}/test/geocoding")
file (GLOB_RECURSE GEOCODING_TEST_SOURCES "${GEOCODING_TEST_DIR}/*.txt")

set (GEOCODING_TEST_DATA_OUTPUT
  "${CMAKE_SOURCE_DIR}/test/phonenumbers/geocoding/geocoding_test_data.cc"
)

add_custom_command (
  COMMAND generate_geocoding_data "${GEOCODING_TEST_DIR}"
    "${GEOCODING_TEST_DATA_OUTPUT}" "_test"

  OUTPUT ${GEOCODING_TEST_DATA_OUTPUT}
  DEPENDS ${GEOCODING_TEST_SOURCES} generate_geocoding_data
  COMMENT "Generating geocoding test data code"
)

set (TEST_SOURCES
  "test/phonenumbers/asyoutypeformatter_test.cc"
  "test/phonenumbers/geocoding/area_code_map_test.cc"
  "test/phonenumbers/geocoding/geocoding_data_test.cc"
  "test/phonenumbers/geocoding/geocoding_test_data.cc"
  "test/phonenumbers/geocoding/mapping_file_provider_test.cc"
  "test/phonenumbers/logger_test.cc"
  "test/phonenumbers/phonenumberutil_test.cc"
  "test/phonenumbers/regexp_adapter_test.cc"
  "test/phonenumbers/regexp_cache_test.cc"
  "test/phonenumbers/run_tests.cc"
  "test/phonenumbers/shortnumberutil_test.cc"
  "test/phonenumbers/stringutil_test.cc"
  "test/phonenumbers/test_util.cc"
  "test/phonenumbers/unicodestring_test.cc"
  "test/phonenumbers/utf/unicodetext_test.cc"
)
if (${USE_ICU_REGEXP} STREQUAL "ON")
  # Add the phone number matcher tests.
  list (APPEND TEST_SOURCES "test/phonenumbers/phonenumbermatch_test.cc")
  list (APPEND TEST_SOURCES "test/phonenumbers/phonenumbermatcher_test.cc")
endif ()

# Build the testing binary.
include_directories ("test")
add_executable (libphonenumber_test ${TEST_SOURCES})
set (TEST_LIBS phonenumber_testing ${GTEST_LIB})

if (NOT WIN32)
  list (APPEND TEST_LIBS pthread)
endif ()

target_link_libraries (libphonenumber_test ${TEST_LIBS})
add_custom_target (test
  COMMAND generate_geocoding_data_test
  COMMAND libphonenumber_test

  DEPENDS generate_geocoding_data_test libphonenumber_test
)

# Install rules.
install (FILES
  "src/phonenumbers/asyoutypeformatter.h"
  "src/phonenumbers/callback.h"
  "src/phonenumbers/logger.h"
  "src/phonenumbers/phonenumber.pb.h"
  "src/phonenumbers/phonemetadata.pb.h"
  "src/phonenumbers/phonenumberutil.h"
  "src/phonenumbers/regexp_adapter.h"
  "src/phonenumbers/regexp_cache.h"
  "src/phonenumbers/shortnumberutil.h"
  "src/phonenumbers/unicodestring.h"
  DESTINATION include/phonenumbers/
)

install (FILES "src/phonenumbers/utf/unicodetext.h"
         DESTINATION include/phonenumbers/utf/)

if (${USE_ICU_REGEXP} STREQUAL "ON")
  # Install the phone number matcher headers.
  install (FILES
    "src/phonenumbers/phonenumbermatch.h"
    "src/phonenumbers/phonenumbermatcher.h"
    "src/phonenumbers/regexp_adapter.h"
    DESTINATION include/phonenumbers/
  )
endif ()

install (FILES "src/base/basictypes.h" DESTINATION include/base/)

install (FILES
  "src/base/memory/scoped_ptr.h"
  "src/base/memory/singleton.h"
  DESTINATION include/base/memory/
)

install (FILES "src/base/synchronization/lock.h"
         DESTINATION include/base/synchronization/)

install (TARGETS phonenumber LIBRARY DESTINATION lib/ ARCHIVE DESTINATION lib/)

if (BUILD_SHARED_LIB)
  install (TARGETS phonenumber-shared LIBRARY DESTINATION lib/ ARCHIVE
           DESTINATION lib/)
endif ()
