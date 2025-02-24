#
#    Copyright (c) 2020 Project CHIP Authors
#
#    Licensed under the Apache License, Version 2.0 (the "License");
#    you may not use this file except in compliance with the License.
#    You may obtain a copy of the License at
#
#        http://www.apache.org/licenses/LICENSE-2.0
#
#    Unless required by applicable law or agreed to in writing, software
#    distributed under the License is distributed on an "AS IS" BASIS,
#    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#    See the License for the specific language governing permissions and
#    limitations under the License.
#
cmake_minimum_required(VERSION 3.13.1)

get_filename_component(CHIP_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/third_party/connectedhomeip REALPATH)
get_filename_component(NRFCONNECT_COMMON ${CHIP_ROOT}/examples/platform/nrfconnect REALPATH)
get_filename_component(APP_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/.. REALPATH)
get_filename_component(CHEF ${CMAKE_CURRENT_SOURCE_DIR}/../ REALPATH)

include(${CHIP_ROOT}/config/nrfconnect/app/check-nrfconnect-version.cmake)
include(${CHEF}/project_include.cmake)

get_filename_component(GEN_DIR ${CHEF}/out/${SAMPLE_NAME}/zap-generated REALPATH)

set(CONF_FILE prj.conf)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CHIP_CFLAGS "${CHIP_CFLAGS} -DCHIP_DEVICE_CONFIG_DEVICE_PRODUCT_ID=${CONFIG_DEVICE_PRODUCT_ID}")
set(CHIP_CFLAGS "${CHIP_CFLAGS} -DCHIP_DEVICE_CONFIG_DEVICE_VENDOR_ID=${CONFIG_DEVICE_VENDOR_ID}")
set(CHIP_CFLAGS "${CHIP_CFLAGS} -DCHIP_PLATFORM_NRFCONNECT=1")
message(STATUS "Product ID " ${CONFIG_DEVICE_PRODUCT_ID})
message(STATUS "Vendor ID " ${CONFIG_DEVICE_VENDOR_ID})

# Set Kconfig root files that will be processed as a first Kconfig for used child images.
set(mcuboot_KCONFIG_ROOT ${CHIP_ROOT}/config/nrfconnect/chip-module/Kconfig.mcuboot.root)
set(multiprotocol_rpmsg_KCONFIG_ROOT ${CHIP_ROOT}/config/nrfconnect/chip-module/Kconfig.multiprotocol_rpmsg.root)

list(APPEND ZEPHYR_EXTRA_MODULES ${CHIP_ROOT}/config/nrfconnect/chip-module)
find_package(Zephyr HINTS $ENV{ZEPHYR_BASE})

if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/boards/${BOARD}.conf)
  list(APPEND CONF_FILE boards/${BOARD}.conf)
endif()

project(chip-nrfconnect-chef-example)

include(${CHIP_ROOT}/config/nrfconnect/app/enable-gnu-std.cmake)
include(${CHIP_ROOT}/src/app/chip_data_model.cmake)

# -Wmaybe-uninitialized has too many false positives, including on std::optional
# and chip::Optional.  Make it nonfatal.
#
# See https://gcc.gnu.org/bugzilla/show_bug.cgi?id=80635
target_compile_options(app PRIVATE -Werror -Wno-error=maybe-uninitialized)

target_include_directories(app PRIVATE
                           ${CMAKE_CURRENT_SOURCE_DIR}
                           ${CMAKE_CURRENT_SOURCE_DIR}/third_party/connectedhomeip/zzz_generated/app-common
                           ${APP_ROOT}/shell_common/include
                           ${GEN_DIR}
                           ${CHEF}
                           ${GEN_DIR}/../
                           ${CHIP_ROOT}/src
                           ${NRFCONNECT_COMMON}/util/include
                           ${NRFCONNECT_COMMON}/app/include
)

target_sources(app PRIVATE
    ${APP_ROOT}/shell_common/globals.cpp
    ${APP_ROOT}/shell_common/cmd_misc.cpp
    ${APP_ROOT}/shell_common/cmd_otcli.cpp
    ${APP_ROOT}/shell_common/cmd_ping.cpp
    ${APP_ROOT}/shell_common/cmd_send.cpp
    ${APP_ROOT}/nrfconnect/main.cpp
    ${GEN_DIR}/callback-stub.cpp
    ${GEN_DIR}/IMClusterCommandHandler.cpp
    ${NRFCONNECT_COMMON}/util/ThreadUtil.cpp
)

message(STATUS ${CHEF}/devices/${SAMPLE_NAME}.zap)
chip_configure_data_model(app
    INCLUDE_SERVER
    ZAP_FILE ${CHEF}/devices/${SAMPLE_NAME}.zap
)


# 1) The example needs be linked with openthread command line interpreter (required by cmd_otcli.cpp)
# 2) Zephyr shell must be disabled not to interfere with the example
# We can't achieve 1 and 2 with Kconfig since CONFIG_OPENTHREAD_SHELL implies CONFIG_SHELL.
# Hence the workaround below.
if(CONFIG_OPENTHREAD_FTD)
    target_link_libraries(app INTERFACE openthread-cli-ftd)
elseif(CONFIG_OPENTHREAD_MTD)
    target_link_libraries(app INTERFACE openthread-cli-mtd)
endif()

include(${CHIP_ROOT}/config/nrfconnect/app/flashing.cmake)
