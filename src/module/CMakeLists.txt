# SPDX-License-Identifier: GPL-2.0
# Copyright (c) 2019-2020 VMware, Inc. All rights reserved.
# Copyright (c) 2016-2019 Carbon Black, Inc. All rights reserved.

cmake_minimum_required(VERSION 3.12)
project (KernelEventCollectorModule)

# LOCAL_BUILD defines if the build will be performed with configuration options
# provided directly in this CMake file, or if the CB build utility will \
# substitute values in.
if(NOT DEFINED LOCAL_BUILD)
    set(LOCAL_BUILD no)
endif()

if(DEFINED ENV{CONTAINER_TOP_DIR})
    set(ROOT_DIR $ENV{CONTAINER_TOP_DIR})
else()
    # Resolves to a real path without the ".."
    get_filename_component(ROOT_DIR "${PROJECT_SOURCE_DIR}/../.." ABSOLUTE)
endif()

if(DEFINED ENV{PLATFORM_BUILD})
    set(PLATFORM_BUILD_DIR $ENV{PLATFORM_BUILD})
else()
    set(PLATFORM_BUILD_DIR "${CMAKE_BINARY_DIR}")
endif()

if(DEFINED ENV{ORIGINAL_SOURCE})
    set(ROOT_SOURCE_DIR $ENV{ORIGINAL_SOURCE})
else()
    # Resolves to a real path without the ".."
    get_filename_component(ROOT_SOURCE_DIR "${PROJECT_SOURCE_DIR}/.." ABSOLUTE)
endif()

include(constants.cmake)

# Variables determined from build type
if (${LOCAL_BUILD})
    message("Building using variables defined locally in this CMake")
    # Include the CbKernelUtil helpers
    set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${ROOT_DIR}/cmake")

    if(NOT KERNEL_NAME)
        message(FATAL_ERROR "You must specify the kernel version to build for with:\n"
                "    -DKERNEL_NAME=<version>.\n"
                "See the README for officially supported kernel versions.\n"
                "To build for the current distribution, use:\n"
                "    -DKERNEL_NAME=$(uname -r)\n")
    endif()

    set(KERNEL_LIST INSERT 0 "${KERNEL_NAME}")
else()
    message("Building using variables provided by CB build utility")
    find_package(CbUtil REQUIRED)
    cb_conan_setup(TARGETS
            CONAN_INCLUDE ${PLATFORM_BUILD_DIR}/conanbuildinfo.cmake)

    # Loop over the conan dependencies and add all the kernels that we find.
    #  Note: I am trusting that conan gives them to me sorted.  I insert each new kernel at the beginning of the list.
    #        This causes the "newest" kernel to be built first.
    foreach(ITEM ${CONAN_DEPENDENCIES})
        if(ITEM MATCHES "Kernel_*")
            list(INSERT KERNEL_LIST 0 "${ITEM}")
        endif()
    endforeach()
endif()

# Check for values from the ENV
if(DEFINED ENV{OPTION_MODULE_NAME})
    set(MODULE_NAME $ENV{OPTION_MODULE_NAME})
else()
    set(MODULE_NAME event_collector)
endif()
string(TOUPPER "${MODULE_NAME}" UPPER_MODULE_NAME)

if(DEFINED ENV{PROJECT_VERSION})
    set(VERSION_STRING $ENV{PROJECT_VERSION})
else()
    set(VERSION_STRING "0.0.0")
endif()


#############################################################################################
# Builds the module version suffix, from the VERSION_STRING.
#  This version suffix becomes a part of the ".ko" filename, also is compiled into code and
#  becomes a part of the device name created by the module.
#  Doing this should allow for more than one kernel-modules to be installed on the system
#  (since each will have its own unique device-node.)
#  example:
#    VERSION_STRING would be 1.6.12349
#    MODULE_VERSION_SUFFIX would be 1_6_12349
#
# Converting dots to underscore just because insmod does not like dots.
#############################################################################################
string(REPLACE "." "_" MODULE_VERSION_SUFFIX ${VERSION_STRING})
set(MODULE_NAME ${MODULE_NAME}_${MODULE_VERSION_SUFFIX})

# These items are calculated based on the configured MODULE_NAME
set(PROC_DIR ${${UPPER_MODULE_NAME}_PROC_DIR})
set(DEBUG_PREFIX ${${UPPER_MODULE_NAME}_DEBUG_PREFIX})
set(MEM_CACHE_PREFIX ${${UPPER_MODULE_NAME}_MEM_CACHE_PREFIX})

string(TIMESTAMP BUILD_DATE "%Y-%m-%d - %H:%M:%S")
set(KERNEL_BUILD_PATH ${PLATFORM_BUILD_DIR}/kernel-builds)

find_package(CbKernelUtil REQUIRED)

# Common variable definitions
set(INCLUDE_DIR $ENV{ORIGINAL_SOURCE}/include)

set(CBSENSOR_AFLAGS "")
set(CBSENSOR_AFLAGS "${CBSENSOR_AFLAGS} -I${PROJECT_SOURCE_DIR}")
set(CBSENSOR_AFLAGS "${CBSENSOR_AFLAGS} -I${PROJECT_BINARY_DIR}")
set(CBSENSOR_AFLAGS "${CBSENSOR_AFLAGS} -I${INCLUDE_DIR}")

set(CBSENSOR_FLAGS "")
set(CBSENSOR_FLAGS "${CBSENSOR_FLAGS} -Wall")
set(CBSENSOR_FLAGS "${CBSENSOR_FLAGS} -Wformat")
set(CBSENSOR_FLAGS "${CBSENSOR_FLAGS} -Werror")
set(CBSENSOR_FLAGS "${CBSENSOR_FLAGS} -I${PROJECT_SOURCE_DIR}")
set(CBSENSOR_FLAGS "${CBSENSOR_FLAGS} -I${PROJECT_SOURCE_DIR}/tests")
set(CBSENSOR_FLAGS "${CBSENSOR_FLAGS} -I${PROJECT_BINARY_DIR}")
set(CBSENSOR_FLAGS "${CBSENSOR_FLAGS} -I${INCLUDE_DIR}")
set(CBSENSOR_FLAGS "${CBSENSOR_FLAGS} -include \"version.h\"")

# Prepare source files (and remove configured files from a previous build)
set(CONFIGURED_VERSION_FILE ${PROJECT_BINARY_DIR}/version.h)
file(REMOVE ${CONFIGURED_VERSION_FILE})
configure_file(${INCLUDE_DIR}/version.h.in ${CONFIGURED_VERSION_FILE})

set(SOURCE_FILES drvmain.c
        hook-tracking.c
        findsyms.c
        page-helpers.c
        path-buffers.c
        mem-cache.c
        rbtree-helper.c
        file-helper.c
        file-hooks.c
        logger.c
        lsm.c
        modules.c
        cb-spinlock.c
        net-helper.c
        net-hooks.c
        process-hooks.c
        task-helper.c
        fops-comm.c
        process-tracking.c
        process-tracking-sorted.c
        process-tracking-discovery.c
        process-tracking-show.c
        process-tracking-helpers.c
        process-tracking-file-cache.c
        cb-isolation.c
        cb-banning.c
        netfilter.c
        hash-table-generic.c
        file-process-tracking.c
        syscall_init.c
        cb-stats.c
        module-hooks.c
        event-factory.c
        cb-module-state-export.c
        dns-parser.c
        tests/run-tests.c
        tests/hashtabl-tests.c
        tests/process-tracking-tests.c
        tests/module-state-tests.c)

foreach(KERNEL_NAME ${KERNEL_LIST})
    if(${LOCAL_BUILD})
        set(KERNEL_VERSION "${KERNEL_NAME}")
        set(KERNEL_BUILD_DIR "/lib/modules/${KERNEL_NAME}/build")
    else()
        string(TOUPPER "${KERNEL_NAME}" UPPER_KERNEL_NAME)
        set(KERNEL_VERSION "${CONAN_USER_${UPPER_KERNEL_NAME}_version}")
        set(KERNEL_BUILD_DIR "${CONAN_RES_DIRS_${UPPER_KERNEL_NAME}}")
    endif()
    cb_add_kernel_module(
            NAME                ${MODULE_NAME}
            KERNEL_NAME         ${KERNEL_NAME}
            KERNEL_VERSION      ${KERNEL_VERSION}
            OUTPUT_PATH         ${KERNEL_BUILD_PATH}
            MODULE_SOURCE_DIR   ${PROJECT_SOURCE_DIR}
            MODULE_BUILD_DIR    ${PROJECT_BINARY_DIR}
            KERNEL_BUILD_DIR    ${KERNEL_BUILD_DIR}
            FLAGS               ${CBSENSOR_FLAGS}
            AFLAGS              ${CBSENSOR_AFLAGS}
            SOURCE_FILES        ${SOURCE_FILES}
            USE_NATIVE_COMPILER)
endforeach()

file (GLOB SRC_FILES
        RELATIVE $ENV{ORIGINAL_SOURCE}
        *.h
        *.c
        )

cb_check_kernel_files(
        SOURCE_DIR          $ENV{ORIGINAL_SOURCE}
        IGNORE_TAGS         CODE_INDENT
        CONSTANT_COMPARISON
        LEADING_SPACE
        LINUX_VERSION_CODE
        NEW_TYPEDEFS
        OPEN_BRACE
        SUSPECT_CODE_INDENT
        TRAILING_STATEMENTS
        AVOID_EXTERNS
        # checkpatch.pl does not like new typedefs.  We possibly should list all the typedefs we add here, but for now
        #  I am only listing the ones that are giving me issues.
        #  If you get the error `need consistent spacing around "*"` then add the type here.
        NEW_TYPES           CB_FILE_TYPE
        ProcessContext
        CB_EVENT_DNS_RESPONSE
        SOURCE_FILES        ${SRC_FILES}
)
