# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

cmake_minimum_required(VERSION 2.6 FATAL_ERROR)

set(CMAKE_BUILD_TYPE, Release)

include(../../../../../hadoop-common-project/hadoop-common/src/JNIFlags.cmake NO_POLICY_SCOPE)

# determine if container-executor.conf.dir is an absolute
# path in case the OS we're compiling on doesn't have
# a hook in get_executable. We'll use this define
# later in the code to potentially throw a compile error
string(REGEX MATCH . HCD_ONE "${HADOOP_CONF_DIR}")
string(COMPARE EQUAL ${HCD_ONE} / HADOOP_CONF_DIR_IS_ABS)

include(CheckIncludeFiles)
check_include_files("sys/types.h;sys/sysctl.h" HAVE_SYS_SYSCTL_H)

include(CheckFunctionExists)
CHECK_FUNCTION_EXISTS(fcloseall HAVE_FCLOSEALL)

include(CheckSymbolExists)
check_symbol_exists(sysctl "sys/types.h;sys/sysctl.h" HAVE_SYSCTL)

if(APPLE)
  include_directories( /System/Library/Frameworks )
  find_library(COCOA_LIBRARY Cocoa)
  mark_as_advanced(COCOA_LIBRARY)
  set(EXTRA_LIBS ${COCOA_LIBRARY})
endif(APPLE)

function(output_directory TGT DIR)
    SET_TARGET_PROPERTIES(${TGT} PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${DIR}")
    SET_TARGET_PROPERTIES(${TGT} PROPERTIES
        ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${DIR}")
    SET_TARGET_PROPERTIES(${TGT} PROPERTIES
        LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${DIR}")
endfunction(output_directory TGT DIR)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -Wall -O2 -D_GNU_SOURCE")
# note: can't enable -D_LARGEFILE: see MAPREDUCE-4258
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_REENTRANT")

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_BINARY_DIR}
    main/native/container-executor
    main/native/container-executor/impl
)
CONFIGURE_FILE(${CMAKE_SOURCE_DIR}/config.h.cmake ${CMAKE_BINARY_DIR}/config.h)

add_library(container
    main/native/container-executor/impl/configuration.c
    main/native/container-executor/impl/container-executor.c
    main/native/container-executor/impl/get_executable.c
)

add_executable(container-executor
    main/native/container-executor/impl/main.c
)
target_link_libraries(container-executor
    container ${EXTRA_LIBS}
)
output_directory(container-executor target/usr/local/bin)

add_executable(test-container-executor
    main/native/container-executor/test/test-container-executor.c
)
target_link_libraries(test-container-executor
    container ${EXTRA_LIBS}
)
output_directory(test-container-executor target/usr/local/bin)
