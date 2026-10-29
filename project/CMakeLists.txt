#
# (c) 2026 Copyright, Real-Time Innovations, Inc.  All rights reserved.
#
# RTI grants Licensee a license to use, modify, compile, and create derivative
# works of the Software.  Licensee has the right to distribute object form
# only for use with RTI products.  The Software is provided "as is", with no
# warranty of any type, including any warranty for fitness for any purpose.
# RTI is under no obligation to maintain or support the Software.  RTI shall
# not be liable for any incidental or consequential damages arising out of the
# use or inability to use the software.
#

cmake_minimum_required(VERSION 3.21)
project(xcompile-shapes)

# Add the folder where the FindRTIConnextDDS.cmake file is placed to the
# CMAKE_MODULE_PATH variable
list(APPEND CMAKE_MODULE_PATH
     "${CMAKE_CURRENT_SOURCE_DIR}/resources/cmake/rticonnextdds-cmake-utils/cmake/Modules"
)
include(FindRTIConnextDDS)

# Find the ConnextDDS libraries. This will look for the core and API libraries
# only
find_package(
    RTIConnextDDS "7.3.0"
    REQUIRED
    COMPONENTS
	core
)

# Run Codegen to generate the source code files for the types
include(ConnextDdsCodegen)
connextdds_rtiddsgen_run(
    IDL_FILE
        "${CMAKE_CURRENT_SOURCE_DIR}/shapes.idl"
    OUTPUT_DIRECTORY
        "${CMAKE_CURRENT_BINARY_DIR}/src"
    LANG C++11
    VAR "SHAPE_TYPE"
    DISABLE_PREPROCESSOR

# The options are DDS_PSM_Cxx for the default mapping in connextdds
# 7.5.0 and earlier. Use IDL4_CPP for the new standard mapping defined by
# the OMG in 2023. When -standard is not specified, the default value is
# IDL4_CPP.    
#    EXTRA_ARGS
#        -standard DDS_PSM_Cxx
)

add_custom_target(codegen_sources
    DEPENDS
        ${shapes_CXX_GENERATED_SOURCES}
)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Executables (each links the common sources)
# Create the target for the publisher executable
add_executable(shapes_publisher
    "${CMAKE_CURRENT_SOURCE_DIR}/shapes_publisher.cxx"
    ${SHAPE_TYPE_CXX11_SOURCES}
)

# Create the target for the publisher executable
add_executable(shapes_subscriber
    "${CMAKE_CURRENT_SOURCE_DIR}/shapes_subscriber.cxx"
    ${SHAPE_TYPE_CXX11_SOURCES}
)

# Link against the ConnextDDS libraries
target_link_libraries(
    shapes_publisher
    PRIVATE
        RTIConnextDDS::cpp2_api
)

target_link_libraries(
    shapes_subscriber
    PRIVATE
        RTIConnextDDS::cpp2_api
)

# Include the folder with the type header files
target_include_directories(shapes_publisher
    PRIVATE
    "${CMAKE_CURRENT_BINARY_DIR}/src"
)

target_include_directories(shapes_subscriber
    PRIVATE
    "${CMAKE_CURRENT_BINARY_DIR}/src"
)
# Copy the USER_QOS_PROFILES.xml
add_custom_target(copy_qos
    DEPENDS
        "${CMAKE_CURRENT_BINARY_DIR}/USER_QOS_PROFILES.xml"
)

add_custom_command(
    OUTPUT
        "${CMAKE_CURRENT_BINARY_DIR}/USER_QOS_PROFILES.xml"
    COMMAND
        ${CMAKE_COMMAND} -E copy_if_different
            "${CMAKE_CURRENT_SOURCE_DIR}/USER_QOS_PROFILES.xml"
            "${CMAKE_CURRENT_BINARY_DIR}"
    COMMENT "Copying USER_QOS_PROFILES.xml"
    DEPENDS
        "${CMAKE_CURRENT_SOURCE_DIR}/USER_QOS_PROFILES.xml"
    VERBATIM
)

# Copy the USER_QOS_PROFILES.XML when the publisher is built and ensure
# Codegen is called
add_dependencies(shapes_publisher
    copy_qos
    codegen_sources
)

# Copy the USER_QOS_PROFILES.XML when the subscriber is built and ensure
# Codegen is called
add_dependencies(shapes_subscriber
    copy_qos
    codegen_sources
)