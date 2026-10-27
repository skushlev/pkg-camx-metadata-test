cmake_minimum_required(VERSION 3.16)

# Define version - can be passed from Yocto via command line
if(NOT DEFINED PROJECT_VERSION)
    set(PROJECT_VERSION "0.1.0")
endif()

project(camx_metadata VERSION ${PROJECT_VERSION})

# Add destination of compiletime pkg-config file to PKG_CONFIG_PATH
set(ENV{PKG_CONFIG_PATH} "${CMAKE_CURRENT_BINARY_DIR}:$ENV{PKG_CONFIG_PATH}")

configure_file(
  camx_metadata.pc.in
  camx_metadata.pc
  @ONLY
)

# Files and Build Type
add_library( camx_metadata SHARED
    camera/src/camera_metadata.c
)

target_include_directories(camx_metadata PRIVATE
    private/camera/include
    camera/include
)

set_target_properties(camx_metadata PROPERTIES
    VERSION ${PROJECT_VERSION}
    SOVERSION ${PROJECT_VERSION_MAJOR}
    OUTPUT_NAME "camx_metadata"
)

install(TARGETS camx_metadata DESTINATION lib)
install(DIRECTORY camera/include/system DESTINATION include)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/camx_metadata.pc DESTINATION lib/pkgconfig)
