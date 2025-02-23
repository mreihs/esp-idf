# For more information about build system see
# https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/build-system.html
# The following five lines of boilerplate have to be in your project's
# CMakeLists in this exact order for cmake to work correctly
cmake_minimum_required(VERSION 3.5)

set(g0_components soc hal esp_common esp_rom)  # also <arch>, i.e. xtensa or riscv, will be added below
set(g1_components spi_flash freertos log heap newlib esp_system esp_hw_support)
set(COMPONENTS ${g0_components} ${g1_components} main)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)

idf_build_set_property(__BUILD_COMPONENT_DEPGRAPH_ENABLED 1)

project(g1_components)

set(extra_allowed_components
    ${CONFIG_IDF_TARGET_ARCH}
    ${CONFIG_IDF_TARGET}
)

# These components are currently included into "G1" build, but shouldn't.
# After removing the extra dependencies, remove the components from this list as well.
set(extra_components_which_shouldnt_be_included
    # app_trace is a public dependency of FreeRTOS because of SystemView trace headers.
    #   To fix, convert this into a weak public dependency conditional on CONFIG_APPTRACE_SV_ENABLE.
    app_trace
    # app_update gets added because of bootloader_support, spi_flash, espcoredump, esp_system.
    #   bootloader_support, spi_flash, espcoredump should be removed from dependencies;
    #   esp_system code that reads app version should be made conditional on app_update being included
    #   (via weak CMake dependency and #if __has_include in C code)
    app_update
    # of G1 components, bootloader is only included from spi_flash
    #   [refactor-todo]: see if this dependency from spi_flash can be made weak
    bootloader
    # bootloader_support is a dependency of efuse, app_update, spi_flash.
    #   efuse and app_update should be removed from G1 build;
    #   [refactor-todo]: see if the dependency from spi_flash can be made weak
    bootloader_support
    # console is pulled in by openthread, which should be removed
    console
    # [refactor-todo]: should cxx be in G1? Can it exist without FreeRTOS?
    cxx
    # [refactor-todo]: driver is a dependency of esp_pm, esp_timer, spi_flash, vfs, esp_wifi, ${IDF_TARGET}
    #   all of these should be removed from G1 except for spi_flash.
    driver
    # [refactor-todo]: efuse is a dependency of esp_hw_support, esp_system.
    #   Figure out if these components can exist without a dependency on efuse.
    #   If not, see if esp_hw_support can provide minimal efuse component replacement in G1 build.
    efuse
    # esp_eth is a dependency of esp_netif, esp_event, lwip
    #   Once they are removed from G1, esp_eth will be removed as well.
    esp_eth
    # esp_event is a dependency of esp_wifi, esp_eth. Both should be removed from G1.
    esp_event
    # esp_gdbstub is a dependency of esp_system, can easily be made conditional on related Kconfig option.
    esp_gdbstub
    # esp_netif is a dependency of lwip and esp_event, should disappear once lwip is removed.
    esp_netif
    # esp_phy is a dependency of esp_system and esp_wifi. For the former, it can be made a weak dependency.
    esp_phy
    # esp_pm is pulled in by esp_system and freertos, can be made a weak dependency
    #   conditional on related Kconfig option. It is also used by esp_wifi, driver, mbedtls,
    #   all of which should be removed from G1-only build.
    esp_pm
    # esp_ringbuf is a dependency of driver, which should be removed.
    esp_ringbuf
    # esp_timer is a dependency of freertos, esp_event, esp_wifi, driver.
    #   For freertos, it can be made a weak dependency conditional on FREERTOS_RUN_TIME_STATS_USING_ESP_TIMER
    esp_timer
    # esp_wifi is a dependency of lwip.
    #   [refactor-todo]: investigate making esp_wifi a conditional dependency.
    esp_wifi
    # Similar to esp_gdbstub, this dependency from esp_system can be made weakly dependent
    #   on CONFIG_ESP_COREDUMP_ENABLE.
    #   [refactor-todo]: how will the user set CONFIG_ESP_COREDUMP_ENABLE if the component
    #   is not included into the build, hence Kconfig option is not visible in menuconfig?
    espcoredump
    # esptool_py is a dependency of bootloader, esp_wifi, app_update, partition_table, all of which
    #   should be removed from G1-only build.
    esptool_py
    # a recent addition to the G1-only build, ieee802154 is a dependency of openthread, to be removed
    #  once openthread is (easily) removed.
    ieee802154
    # lwip is a common component due to "sys/socket.h" header.
    #   [refactor-todo] consider adding a system-level sys/socket.h in newlib instead
    lwip
    # mbedtls is a dependency of bootloader_support (plus other easier-to-remove ones)
    #   it is hard to make it conditional, need to remove bootloader_support.
    mbedtls
    # nvs_flash is required by:
    #   esp_system — no obvious reason, [refactor-todo] why?
    #   lwip — can be turned into a weak dependency
    #   esp_wifi, esp_phy — both should be removed
    nvs_flash
    # openthread is a dependency of lwip, it is easy to turn it into a conditional one
    openthread
    # partition_table is pulled in by app_update, esptool_py, bootloader; all to be removed
    partition_table
    # pthread is required by esp_system (for initialization only, can be made a weak dependency)
    #   and cxx. See also [refactor-todo] about cxx, can it work without pthread?
    pthread
    # tcpip_adapter is a dependency of esp_netif and lwip, both of which should be removed
    tcpip_adapter
    # vfs is a dependency of lwip. It can be made conditional, while lwip is still a common component.
    vfs
    # wpa_supplicant is a dependency of esp_wifi, which is to be removed from G1-only build
    wpa_supplicant
)

set(expected_components
    ${COMPONENTS}
    ${extra_allowed_components}
    ${extra_components_which_shouldnt_be_included}
)

list(SORT expected_components)


idf_build_get_property(build_components BUILD_COMPONENTS)
list(SORT build_components)

if(NOT "${expected_components}" STREQUAL "${build_components}")
    message(FATAL_ERROR "Unexpected components list in G1 build. "
                        "Expected: ${expected_components}. "
                        "Actual: ${build_components}")
endif()
