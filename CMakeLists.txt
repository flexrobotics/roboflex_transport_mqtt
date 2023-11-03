cmake_minimum_required(VERSION 3.18)

project(roboflex_transport_mqtt VERSION 0.1.3 DESCRIPTION "roboflex transport mqtt")

# specify the C++ standard
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED True)


# -------------------- 
# Resolve dependencies

# Include FetchContent Module
include(FetchContent)


# download and build roboflex_core
FetchContent_Declare(roboflex_core
    GIT_REPOSITORY https://github.com/flexrobotics/roboflex.git
    GIT_TAG        main
)
set(BUILD_ROBOFLEX_PYTHON_EXT OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(roboflex_core)

# Include mosquitto. A big mess.
# cmake is no better than bazel for this.
# If anybody knows an easier or better way,
# or maybe using FetchContent, please let me know.
#
# FROM https://github.com/tarantool/mqtt/blob/master/cmake/FindMosquitto.cmake:
#
# - Find libmosquitto
# Find the native libmosquitto includes and libraries
#
#  MOSQUITTO_INCLUDE_DIR - where to find mosquitto.h, etc.
#  MOSQUITTO_LIBRARIES   - List of libraries when using libmosquitto.
#  MOSQUITTO_FOUND       - True if libmosquitto found.


# if (NOT MOSQUITTO_INCLUDE_DIR)
#   find_path(MOSQUITTO_INCLUDE_DIR mosquitto.h)
# endif()

# if (NOT MOSQUITTO_LIBRARY)
#   find_library(
#     MOSQUITTO_LIBRARY
#     NAMES mosquitto)
# endif()

# include(FindPackageHandleStandardArgs)

# find_package_handle_standard_args(
#   MOSQUITTO DEFAULT_MSG
#   MOSQUITTO_LIBRARY MOSQUITTO_INCLUDE_DIR
# )

# message(STATUS "libmosquitto include dir: ${MOSQUITTO_INCLUDE_DIR}")
# message(STATUS "libmosquitto: ${MOSQUITTO_LIBRARY}")
# set(MOSQUITTO_LIBRARIES ${MOSQUITTO_LIBRARY})

# mark_as_advanced(MOSQUITTO_INCLUDE_DIR MOSQUITTO_LIBRARY)


find_package(PkgConfig REQUIRED)
pkg_check_modules(Mosquitto IMPORTED_TARGET libmosquitto REQUIRED)


# -------------------- 
# Define the library

add_library(roboflex_transport_mqtt STATIC
    src/mqtt_nodes.cpp
    include/roboflex_transport_mqtt/mqtt_nodes.h
)

# Set some properties on our library
set_property(TARGET roboflex_transport_mqtt PROPERTY 
    POSITION_INDEPENDENT_CODE ON
)

# Include directories when we compile our library
target_include_directories(roboflex_transport_mqtt PUBLIC 
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include> 
    $<INSTALL_INTERFACE:include>
)

# Link against the necessary libraries
target_link_libraries(roboflex_transport_mqtt PUBLIC 
    roboflex_core 
    mosquitto

    # Unfortunately, we need to link against all these libraries
    # because they are used by roboflex_core, and cmake doesn't
    # seem to be transitivily doing it's thing. Urgh, someone help.
    flatbuffers_util 
    xtensor 
    xsimd 
    xtl
    Eigen3::Eigen
)


# -------------------- 
# Examples

# run_webcam_cpp example
add_executable(pub_sub_0_cpp examples/pub_sub_0_cpp.cpp)
target_link_libraries(pub_sub_0_cpp PRIVATE 
    roboflex_core 
    roboflex_transport_mqtt
)


# -------------------- 
# install

# If you need to install the transport_mqtt library
install(TARGETS roboflex_transport_mqtt 
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    RUNTIME DESTINATION bin
    INCLUDES DESTINATION include
)
#install(FILES transport_mqtt.h DESTINATION include/roboflex_transport_mqtt)
install(DIRECTORY include/roboflex_transport_mqtt
    DESTINATION include
)

install(TARGETS roboflex_transport_mqtt
    EXPORT roboflex_transport_mqttTargets
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    RUNTIME DESTINATION bin
    INCLUDES DESTINATION include
)

# export the targets to a make script
install(EXPORT roboflex_transport_mqttTargets
    FILE roboflex_transport_mqttTargets.cmake
    #NAMESPACE roboflex_core:: # do we want this?
    DESTINATION lib/cmake/roboflex_transport_mqtt # do we want or need cmake/ ?
)

# generate actual config file 
# The generated file will include information 
# about where to find the installed headers, 
# libraries, and any other required files,
# INCLUDING DEPENDENCIES such as xtensor.
include(CMakePackageConfigHelpers)
configure_package_config_file(Config.cmake.in # template
    ${CMAKE_CURRENT_BINARY_DIR}/roboflex_transport_mqttConfig.cmake # output
    INSTALL_DESTINATION lib/cmake/roboflex_transport_mqtt # relative to CMAKE_INSTALL_PREFIX
)
# install the config file
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/roboflex_transport_mqttConfig.cmake
    DESTINATION lib/cmake/roboflex_transport_mqtt
)

# --------------------
# build python bindings

add_subdirectory(python)
