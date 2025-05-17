cmake_minimum_required(VERSION 3.8)
project(MediaInfo_CLI VERSION 1.0.0 LANGUAGES CXX)

# Set C++ standard
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Include directories
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../../../Source)

# Source files
file(GLOB CLI_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/../../../Source/CLI/*.cpp)

# Find required packages
find_package(ZLIB REQUIRED)
find_package(Threads REQUIRED)
find_package(zenlib REQUIRED)
find_package(tinyxml2 REQUIRED)
find_package(mediainfolib REQUIRED)

# Main executable
add_executable(mediainfo_cli ${CLI_SOURCES})
target_compile_definitions(mediainfo_cli PRIVATE UNICODE)

# Link libraries
target_link_libraries(mediainfo_cli
    PRIVATE
    mediainfo
    tinyxml2::tinyxml2
    ZLIB::ZLIB
    Threads::Threads
)

install(TARGETS mediainfo_cli DESTINATION bin)
