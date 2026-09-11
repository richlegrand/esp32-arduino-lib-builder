set(BITBANG_REQUIRES "")
if(BITBANG_AVAILABLE)
    # Pulls the rest of the stack in transitively: kvs_webrtc, esp_usrsctp,
    # libsrtp2, kvs_utils, and the signaling and bridge components.
    set(BITBANG_REQUIRES app_webrtc bitbang_httpd bitbang_signaling)
endif()

idf_component_register(SRCS "sketch.cpp" "arduino-lib-builder-gcc.c" "arduino-lib-builder-cpp.cpp" "arduino-lib-builder-as.S"
                       INCLUDE_DIRS "."
                       REQUIRES ${BITBANG_REQUIRES})
