# The following lines of boilerplate have to be in your project's
# CMakeLists in this exact order for cmake to work correctly
cmake_minimum_required(VERSION 3.5)

# BitBang: the WebRTC stack is built into the precompiled libraries, the same
# way mbedTLS and lwIP are. An Arduino library is only source compiled by the
# Arduino builder -- it has no CMake and no IDF component system -- so it cannot
# build usrsctp and the KVS SDK itself. Override either path with an env var.
if(DEFINED ENV{BITBANG_KVS_PORT})
    set(BITBANG_KVS_PORT "$ENV{BITBANG_KVS_PORT}")
else()
    set(BITBANG_KVS_PORT "${CMAKE_CURRENT_SOURCE_DIR}/../kvs-webrtc-esp/esp_port")
endif()
if(DEFINED ENV{BITBANG_ESP32})
    set(BITBANG_ESP32 "$ENV{BITBANG_ESP32}")
else()
    set(BITBANG_ESP32 "${CMAKE_CURRENT_SOURCE_DIR}/../bitbang-esp32")
endif()
# IDF compiles every component it discovers in a component directory, whatever
# the requires graph says, so pointing at these two trees pulls in ten we never
# link -- and two of those break this build. libwebsockets trips
# -Werror=discarded-qualifiers, and esp_hosted/network_coprocessor inject a
# global ESP_IDF_LWIP_HOOK_FILENAME that reaches every translation unit and
# breaks arduino's own esp32-hal-hosted.c.
#
# EXCLUDE_COMPONENTS drops them by name. Symlinking a curated subset does not
# work: these components derive their source root by walking up from their own
# directory, and CMake uses the symlink path, so the root lands somewhere else.
#
# The kept set is from the linker map of a working camera build -- an archive
# with references is needed, one with none is not.
if(EXISTS "${BITBANG_KVS_PORT}/components" AND EXISTS "${BITBANG_ESP32}/components")
    list(APPEND EXTRA_COMPONENT_DIRS
        "${BITBANG_KVS_PORT}/components"
        "${BITBANG_ESP32}/components")
    list(APPEND EXCLUDE_COMPONENTS
        libwebsockets kvs_signaling esp_hosted network_coprocessor
        esp_wifi_remote esp32_p4_function_ev_board media_stream
        webrtc_bridge signaling_serializer credential)
    set(BITBANG_AVAILABLE TRUE)
    message(STATUS "BitBang components: ${BITBANG_KVS_PORT}/components (10 excluded)")
else()
    message(STATUS "BitBang components not found -- building without them")
endif()

include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(arduino-lib-builder)

idf_build_get_property(elf EXECUTABLE GENERATOR_EXPRESSION)

add_custom_command(
	OUTPUT "idf_libs" 
	COMMAND ${CMAKE_SOURCE_DIR}/tools/copy-libs.sh ${IDF_TARGET} "$ENV{CHIP_VARIANT}" "${CONFIG_LIB_BUILDER_FLASHMODE}" "${CONFIG_SPIRAM_MODE_OCT}" "${CONFIG_IDF_TARGET_ARCH_XTENSA}"
	DEPENDS ${elf}
	WORKING_DIRECTORY ${CMAKE_SOURCE_DIR} 
	VERBATIM
)
add_custom_target(idf-libs DEPENDS "idf_libs")

add_custom_command(
	OUTPUT "copy_bootloader" 
	COMMAND ${CMAKE_SOURCE_DIR}/tools/copy-bootloader.sh ${IDF_TARGET} "$ENV{CHIP_VARIANT}" "${CONFIG_LIB_BUILDER_FLASHMODE}" "${CONFIG_LIB_BUILDER_FLASHFREQ}"
	DEPENDS bootloader 
	WORKING_DIRECTORY ${CMAKE_SOURCE_DIR} 
	VERBATIM
)
add_custom_target(copy-bootloader DEPENDS "copy_bootloader")

add_custom_command(
	OUTPUT "mem_variant" 
	COMMAND ${CMAKE_SOURCE_DIR}/tools/copy-mem-variant.sh ${IDF_TARGET} "$ENV{CHIP_VARIANT}" "${CONFIG_LIB_BUILDER_FLASHMODE}" "${CONFIG_SPIRAM_MODE_OCT}"
	DEPENDS ${elf}
	WORKING_DIRECTORY ${CMAKE_SOURCE_DIR} 
	VERBATIM
)
add_custom_target(mem-variant DEPENDS "mem_variant")

idf_build_set_property(COMPILE_DEFINITIONS "-DESP32_ARDUINO_LIB_BUILDER" APPEND)

##################
### ESP Matter ###
##################
idf_build_set_property(CXX_COMPILE_OPTIONS "-std=gnu++2a;-DCHIP_HAVE_CONFIG_H" APPEND)

# esp_matter is not pulled in for all targets (see main/idf_component.yml rules).
idf_build_get_property(build_components BUILD_COMPONENTS)
if("espressif__esp_matter" IN_LIST build_components)
  # GCC 14 + C++20: ClosureControl structs need same-type operator== for std::optional.
  file(TO_CMAKE_PATH "${CMAKE_SOURCE_DIR}/patches/matter_closure_patch.h" MATTER_CLOSURE_PATCH_H)
  string(REPLACE "\\" "/" MATTER_CLOSURE_PATCH_H "${MATTER_CLOSURE_PATCH_H}")
  idf_component_get_property(esp_matter_lib espressif__esp_matter COMPONENT_LIB)
  target_compile_options(${esp_matter_lib} PRIVATE
      "SHELL:-include \"${MATTER_CLOSURE_PATCH_H}\"")
endif()
