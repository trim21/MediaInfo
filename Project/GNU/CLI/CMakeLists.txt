cmake_minimum_required(VERSION 3.10)
project(mediainfo VERSION 25.04 LANGUAGES C CXX)

# Set C++ standard
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Options from configure.ac
option(ENABLE_DEBUG "Enable full debug" OFF)
option(ENABLE_GPROF "Enable gprof log" OFF)
option(ENABLE_UNIVERSAL_BINARY "Create Mac PPC/i386 Universal binary" OFF)
option(ENABLE_ARCH_PPC "Create Mac PPC architecture" OFF)
option(ENABLE_ARCH_PPC64 "Create Mac PPC64 architecture" OFF)
option(ENABLE_ARCH_I386 "Create Mac i386 architecture" OFF)
option(ENABLE_ARCH_X86_64 "Create Mac x86_64 architecture" OFF)
option(ENABLE_ARCH_ARM64 "Create Mac arm64 architecture" OFF)
option(ENABLE_STATICLIBS "Use static libmediainfo and libzen" OFF)
option(ENABLE_STREAM_MISSING "Disable stream support (old compilers)" OFF)
option(WITH_DLL "Do not link to libmediainfo, dynamic loading" OFF)

set(WITH_LIBMEDIAINFO_NAME "mediainfo" CACHE STRING "Force libmediainfo filename")
set(WITH_MACOSX_SDK "" CACHE STRING "Force the Mac SDK")
set(WITH_MACOSX_VERSION_MIN "" CACHE STRING "Force the Mac Version")

# Determine Unicode support from libzen (simplified, assuming libzen handles this)
# In a real scenario, you might need to find_package(ZenLib) and check its properties
# For now, let's add an option, similar to how configure.ac determines it.
option(ENABLE_UNICODE "Enable Unicode support (requires libzen with Unicode)" ON)

if(ENABLE_UNICODE)
    add_compile_definitions(UNICODE)
endif()

if(WITH_DLL)
    add_compile_definitions(MEDIAINFO_DLL_RUNTIME)
    if(NOT WIN32)
        # On Unix-like systems, for dlopen, dlsym, etc.
        find_package(unofficial-dlfcn CONFIG) # CMake module for dlfcn
        if (unofficial-dlfcn_FOUND)
            list(APPEND CMAKE_REQUIRED_LIBRARIES unofficial::dlfcn::dlfcn)
        else()
            # Fallback for systems where dlfcn is part of libc or needs -ldl
            list(APPEND CMAKE_REQUIRED_LIBRARIES dl)
        endif()
    endif()
endif()

# Check for sstream support (simplified)
include(CheckCXXSourceCompiles)
if(ENABLE_UNICODE)
    check_cxx_source_compiles("#include <sstream>
        int main () {std::wstringstream message; message<<1.0; return 0;}" HAVE_WSTRINGSTREAM_SUPPORT)
    if(NOT HAVE_WSTRINGSTREAM_SUPPORT)
        set(ENABLE_STREAM_MISSING ON)
    endif()
else()
    check_cxx_source_compiles("#include <sstream>
        int main () {std::stringstream message; message<<1.0; return 0;}" HAVE_STRINGSTREAM_SUPPORT)
    if(NOT HAVE_STRINGSTREAM_SUPPORT)
        set(ENABLE_STREAM_MISSING ON)
    endif()
endif()

if(ENABLE_STREAM_MISSING)
    add_compile_definitions(STREAM_MISSING)
endif()

# Compiler flags
if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_CLANG)
    if(ENABLE_DEBUG)
        set(CMAKE_C_FLAGS_DEBUG "-O0 -g -Wall -Wundef")
        set(CMAKE_CXX_FLAGS_DEBUG "-O0 -g -Wall -W -Wundef -Wcast-qual -Wno-ctor-dtor-privacy")
    else()
        set(CMAKE_C_FLAGS_RELEASE "-O2")
        set(CMAKE_CXX_FLAGS_RELEASE "-O2")
    endif()
    if(ENABLE_GPROF)
        add_compile_options(-pg)
        add_link_options(-pg)
    endif()
endif()

# Source files
# The main source file is specified by AC_CONFIG_SRCDIR
# Adjust the path relative to this CMakeLists.txt
set(MEDIAINFO_CLI_SOURCES
    ../../../Source/CLI/CLI_Main.cpp
    # Add other source files from the original Makefile.am or inferred sources
)

# Include directories
# Assuming ZenLib and MediaInfoLib headers are needed
# These paths are relative to this CMakeLists.txt
include_directories(
    ../../../Source/CLI
    ../../../../MediaInfoLib/Source
    ../../../../ZenLib/Source
)


# Dependencies: libmediainfo and libzen
# This is a simplified approach. A more robust solution would use find_package
# or add_subdirectory if they are part of the same CMake build.

set(LIBMEDIAINFO_PROJECT_DIR "../../../../MediaInfoLib/Project/GNU/Library")
set(LIBZEN_PROJECT_DIR "../../../../ZenLib/Project/GNU/Library")

if(NOT WITH_DLL)
    if(EXISTS "${LIBMEDIAINFO_PROJECT_DIR}/lib${WITH_LIBMEDIAINFO_NAME}.la") # .la is libtool archive, check for CMake equivalent or built lib
        # Assuming MediaInfoLib has a CMake build or provides a config file
        # This part needs to be adapted based on how MediaInfoLib is built/found
        message(STATUS "Using MediaInfoLib from local build.")
        # add_subdirectory(${LIBMEDIAINFO_PROJECT_DIR} ${CMAKE_BINARY_DIR}/mediainfolib_build)
        # target_link_libraries(mediainfo_cli PRIVATE mediainfo) # Assuming target name from MediaInfoLib's CMake
        if(ENABLE_STATICLIBS)
            message(STATUS "Linking MediaInfoLib statically (local).")
            # Adjust linking for static library
        else()
            message(STATUS "Linking MediaInfoLib dynamically (local).")
        endif()
    elseif(COMMAND libmediainfo-config)
        message(STATUS "Using MediaInfoLib from system (libmediainfo-config).")
        execute_process(COMMAND libmediainfo-config --cflags OUTPUT_VARIABLE MEDIAINFO_CFLAGS)
        execute_process(COMMAND libmediainfo-config ${IF(ENABLE_STATICLIBS, LIBS_Static, LIBS)} OUTPUT_VARIABLE MEDIAINFO_LIBS)
        string(STRIP "${MEDIAINFO_CFLAGS}" MEDIAINFO_CFLAGS)
        string(STRIP "${MEDIAINFO_LIBS}" MEDIAINFO_LIBS)
        include_directories(${MEDIAINFO_CFLAGS})
        link_libraries(${MEDIAINFO_LIBS})
    elseif(${CMAKE_COMMAND} -E env PKG_CONFIG_PATH=$ENV{PKG_CONFIG_PATH} pkg-config --exists libmediainfo)
        message(STATUS "Using MediaInfoLib from system (pkg-config).")
        find_package(PkgConfig REQUIRED)
        pkg_check_modules(MEDIAINFO REQUIRED libmediainfo)
        include_directories(${MEDIAINFO_INCLUDE_DIRS})
        if(ENABLE_STATICLIBS)
            # pkg-config might need --static for static libs
            pkg_get_variable(MEDIAINFO_STATIC_LIBS libmediainfo Libs_Static)
            if(MEDIAINFO_STATIC_LIBS)
                 link_libraries(${MEDIAINFO_STATIC_LIBS})
            else()
                 link_libraries(${MEDIAINFO_LIBRARIES}) # Fallback or pkg-config --static --libs
            endif()
        else()
            link_libraries(${MEDIAINFO_LIBRARIES})
        endif()
    else()
        message(FATAL_ERROR "libmediainfo configuration is not found. Please install it or ensure it's discoverable.")
    endif()
endif()

# ZenLib dependency (similar logic to MediaInfoLib)
if(EXISTS "${LIBZEN_PROJECT_DIR}/libzen.la")
    message(STATUS "Using ZenLib from local build.")
    # add_subdirectory(${LIBZEN_PROJECT_DIR} ${CMAKE_BINARY_DIR}/zenlib_build)
    # target_link_libraries(mediainfo_cli PRIVATE zen) # Assuming target name from ZenLib's CMake
    if(ENABLE_STATICLIBS)
        message(STATUS "Linking ZenLib statically (local).")
    else()
        message(STATUS "Linking ZenLib dynamically (local).")
    endif()
elif(COMMAND libzen-config)
    message(STATUS "Using ZenLib from system (libzen-config).")
    execute_process(COMMAND libzen-config --cflags OUTPUT_VARIABLE ZENLIB_CFLAGS)
    execute_process(COMMAND libzen-config ${IF(ENABLE_STATICLIBS, LIBS_Static, LIBS)} OUTPUT_VARIABLE ZENLIB_LIBS)
    string(STRIP "${ZENLIB_CFLAGS}" ZENLIB_CFLAGS)
    string(STRIP "${ZENLIB_LIBS}" ZENLIB_LIBS)
    include_directories(${ZENLIB_CFLAGS})
    link_libraries(${ZENLIB_LIBS})
elseif(${CMAKE_COMMAND} -E env PKG_CONFIG_PATH=$ENV{PKG_CONFIG_PATH} pkg-config --exists libzen)
    message(STATUS "Using ZenLib from system (pkg-config).")
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(ZENLIB REQUIRED libzen)
    include_directories(${ZENLIB_INCLUDE_DIRS})
    if(ENABLE_STATICLIBS)
        pkg_get_variable(ZENLIB_STATIC_LIBS libzen Libs_Static)
        if(ZENLIB_STATIC_LIBS)
            link_libraries(${ZENLIB_STATIC_LIBS})
        else()
            link_libraries(${ZENLIB_LIBRARIES}) # Fallback
        endif()
    else()
        link_libraries(${ZENLIB_LIBRARIES})
    endif()
else()
    message(FATAL_ERROR "libzen configuration is not found. Please install it or ensure it's discoverable.")
endif()


# macOS specific settings
if(CMAKE_SYSTEM_NAME MATCHES "Darwin")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -framework CoreFoundation")

    set(OSX_ARCHITECTURES "")
    if(ENABLE_UNIVERSAL_BINARY)
        list(APPEND OSX_ARCHITECTURES "ppc;i386") # Adjust as needed for modern universal binaries (e.g., x86_64;arm64)
    else()
        if(ENABLE_ARCH_PPC)
            list(APPEND OSX_ARCHITECTURES "ppc")
        endif()
        if(ENABLE_ARCH_PPC64)
            list(APPEND OSX_ARCHITECTURES "ppc64")
        endif()
        if(ENABLE_ARCH_I386)
            list(APPEND OSX_ARCHITECTURES "i386")
        endif()
        if(ENABLE_ARCH_X86_64)
            list(APPEND OSX_ARCHITECTURES "x86_64")
        endif()
        if(ENABLE_ARCH_ARM64)
            list(APPEND OSX_ARCHITECTURES "arm64")
        endif()
    endif()

    if(OSX_ARCHITECTURES)
        string(REPLACE ";" " " OSX_ARCHITECTURES_STR "${OSX_ARCHITECTURES}")
        set(CMAKE_OSX_ARCHITECTURES "${OSX_ARCHITECTURES}" CACHE STRING "Build architectures for macOS" FORCE)
        message(STATUS "macOS architectures: ${CMAKE_OSX_ARCHITECTURES}")
    endif()

    if(WITH_MACOSX_SDK AND NOT WITH_MACOSX_SDK STREQUAL "no")
      set(CMAKE_OSX_SYSROOT ${WITH_MACOSX_SDK} CACHE PATH "macOS SDK path" FORCE)
      message(STATUS "macOS SDK: ${CMAKE_OSX_SYSROOT}")
    endif()

    if(WITH_MACOSX_VERSION_MIN AND NOT WITH_MACOSX_VERSION_MIN STREQUAL "no")
      set(CMAKE_OSX_DEPLOYMENT_TARGET ${WITH_MACOSX_VERSION_MIN} CACHE STRING "macOS deployment target" FORCE)
      message(STATUS "macOS deployment target: ${CMAKE_OSX_DEPLOYMENT_TARGET}")
    endif()
endif()

# Add executable
add_executable(mediainfo_cli ${MEDIAINFO_CLI_SOURCES})

# Link libraries found via pkg-config or libX-config earlier
if(MEDIAINFO_LIBS)
    target_link_libraries(mediainfo_cli PRIVATE ${MEDIAINFO_LIBS})
endif()
if(ZENLIB_LIBS)
    target_link_libraries(mediainfo_cli PRIVATE ${ZENLIB_LIBS})
endif()
if(CMAKE_REQUIRED_LIBRARIES) # For -ldl
    target_link_libraries(mediainfo_cli PRIVATE ${CMAKE_REQUIRED_LIBRARIES})
endif()


# Installation (optional, basic example)
# install(TARGETS mediainfo_cli DESTINATION bin)

# Print configuration summary (similar to configure.ac output)
message(STATUS "")
message(STATUS "Configured MediaInfo CLI for '${CMAKE_SYSTEM_NAME}'")
message(STATUS "")
message(STATUS "  Unicode?                                                ${ENABLE_UNICODE}")
message(STATUS "  stream missing support?                                 ${ENABLE_STREAM_MISSING}")
message(STATUS "")
# More detailed status for libzen and libmediainfo would require more complex logic
# to track how they were found (builtin, system, static/dynamic)
message(STATUS "  Using libzen?                                           (see above messages for details)")
message(STATUS "  Using libmediainfo?                                     (see above messages for details)")
message(STATUS "")
message(STATUS "  CMAKE_CXX_FLAGS: ${CMAKE_CXX_FLAGS} ${CMAKE_CXX_FLAGS_${CMAKE_BUILD_TYPE_UPPERCASE}}")
# CMAKE_EXE_LINKER_FLAGS might be more appropriate than CMAKE_SHARED_LINKER_FLAGS for LIBS
message(STATUS "  LINK_LIBRARIES: (see target_link_libraries calls and above messages)")

