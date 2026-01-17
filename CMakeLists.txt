cmake_minimum_required(VERSION 3.15)

# Project definition
project(lcms2 
    VERSION 2.17
    LANGUAGES C CXX
    DESCRIPTION "A free, open source, CMM engine for ICC profiles"
)

# Options
option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(BUILD_STATIC_LIBS "Build static libraries" ON)
option(BUILD_TESTS "Build test programs" ON)
option(BUILD_UTILS "Build utility programs" OFF)
option(BUILD_PLUGINS "Build plugins" OFF)

# Version information
set(LCMS2_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
set(LCMS2_VERSION_MINOR ${PROJECT_VERSION_MINOR})
set(LCMS2_VERSION_MICRO ${PROJECT_VERSION_PATCH})
set(LCMS2_VERSION ${PROJECT_VERSION})

# Library version for shared objects
set(LIBRARY_VERSION "${LCMS2_VERSION_MAJOR}.0.${LCMS2_VERSION_MINOR}")
set(LIBRARY_SOVERSION ${LCMS2_VERSION_MAJOR})

# Check for platform and compiler features
include(CheckIncludeFile)
include(CheckFunctionExists)
include(CheckCCompilerFlag)
include(TestBigEndian)

# Endianness check
test_big_endian(IS_BIG_ENDIAN)

# Compiler definitions
set(COMPILE_DEFINITIONS "")

if(IS_BIG_ENDIAN)
    list(APPEND COMPILE_DEFINITIONS WORDS_BIGENDIAN=1)
endif()

# Visibility attributes for GCC/Clang
if(NOT MSVC)
    check_c_compiler_flag("-fvisibility=hidden" HAS_VISIBILITY_HIDDEN)
    if(HAS_VISIBILITY_HIDDEN)
        list(APPEND COMPILE_DEFINITIONS HAVE_FUNC_ATTRIBUTE_VISIBILITY=1)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fvisibility=hidden")
    endif()
endif()

# Windows DLL support
if(WIN32 AND BUILD_SHARED_LIBS)
    list(APPEND COMPILE_DEFINITIONS CMS_DLL_BUILD=1)
endif()

# Generate version header if needed
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/include/lcms2.h.in")
    configure_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/include/lcms2.h.in
        ${CMAKE_CURRENT_BINARY_DIR}/include/lcms2.h
        @ONLY
    )
endif()

# Source files for the library
set(LCMS2_SRCS
    src/cmsalpha.c
    src/cmscam02.c
    src/cmscgats.c
    src/cmscnvrt.c
    src/cmserr.c
    src/cmsgamma.c
    src/cmsgmt.c
    src/cmshalf.c
    src/cmsintrp.c
    src/cmsio0.c
    src/cmsio1.c
    src/cmslut.c
    src/cmsmd5.c
    src/cmsmtrx.c
    src/cmsnamed.c
    src/cmsopt.c
    src/cmspack.c
    src/cmspcs.c
    src/cmsplugin.c
    src/cmsps2.c
    src/cmssamp.c
    src/cmssm.c
    src/cmstypes.c
    src/cmsvirt.c
    src/cmswtpnt.c
    src/cmsxform.c
)

# Windows resource file
if(WIN32 AND MSVC)
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/Projects/VC2019/lcms2.rc")
        list(APPEND LCMS2_SRCS Projects/VC2019/lcms2.rc)
    endif()
endif()

# Header files
set(LCMS2_HEADERS
    include/lcms2.h
    include/lcms2_plugin.h
)

# Build shared library
if(BUILD_SHARED_LIBS)
    add_library(lcms2 SHARED ${LCMS2_SRCS})
    
    target_compile_definitions(lcms2 PRIVATE ${COMPILE_DEFINITIONS})
    
    target_include_directories(lcms2
        PUBLIC
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
            $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
            $<INSTALL_INTERFACE:include>
        PRIVATE
            ${CMAKE_CURRENT_SOURCE_DIR}/src
    )
    
    set_target_properties(lcms2 PROPERTIES
        VERSION ${LIBRARY_VERSION}
        SOVERSION ${LIBRARY_SOVERSION}
        PUBLIC_HEADER "${LCMS2_HEADERS}"
        C_STANDARD 99
        C_STANDARD_REQUIRED ON
    )
    
    # macOS framework support
    if(APPLE)
        set_target_properties(lcms2 PROPERTIES
            FRAMEWORK FALSE
            MACOSX_RPATH ON
            INSTALL_NAME_DIR "@rpath"
        )
    endif()
    
    # Link with math library on Unix
    if(UNIX AND NOT APPLE)
        target_link_libraries(lcms2 PRIVATE m)
    endif()
endif()

# Build static library
if(BUILD_STATIC_LIBS)
    add_library(lcms2_static STATIC ${LCMS2_SRCS})
    
    target_compile_definitions(lcms2_static PRIVATE ${COMPILE_DEFINITIONS})
    
    target_include_directories(lcms2_static
        PUBLIC
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
            $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
            $<INSTALL_INTERFACE:include>
        PRIVATE
            ${CMAKE_CURRENT_SOURCE_DIR}/src
    )
    
    set_target_properties(lcms2_static PROPERTIES
        OUTPUT_NAME lcms2
        PUBLIC_HEADER "${LCMS2_HEADERS}"
        C_STANDARD 99
        C_STANDARD_REQUIRED ON
    )
    
    # Ensure static library has different name on Windows
    if(WIN32 AND BUILD_SHARED_LIBS)
        set_target_properties(lcms2_static PROPERTIES
            OUTPUT_NAME lcms2_static
        )
    endif()
    
    # Link with math library on Unix
    if(UNIX AND NOT APPLE)
        target_link_libraries(lcms2_static PRIVATE m)
    endif()
endif()

# Build test programs
if(BUILD_TESTS AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/testbed/CMakeLists.txt")
    add_subdirectory(testbed)
endif()

# Build utilities
if(BUILD_UTILS AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/utils/CMakeLists.txt")
    add_subdirectory(utils)
endif()

# Build plugins
if(BUILD_PLUGINS AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/plugins/CMakeLists.txt")
    add_subdirectory(plugins)
endif()

# Installation
include(GNUInstallDirs)

# Install libraries
if(BUILD_SHARED_LIBS)
    install(TARGETS lcms2
        EXPORT lcms2Targets
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )
endif()

if(BUILD_STATIC_LIBS)
    install(TARGETS lcms2_static
        EXPORT lcms2Targets
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )
endif()

# Install headers separately to ensure they're always installed
install(FILES ${LCMS2_HEADERS}
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

# Generate and install pkg-config file
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/lcms2.pc.in")
    configure_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/lcms2.pc.in
        ${CMAKE_CURRENT_BINARY_DIR}/lcms2.pc
        @ONLY
    )
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/lcms2.pc
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
    )
endif()

# Export targets
install(EXPORT lcms2Targets
    FILE lcms2Targets.cmake
    NAMESPACE lcms2::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/lcms2
)

# Generate and install CMake config files
include(CMakePackageConfigHelpers)

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/cmake/lcms2Config.cmake.in")
    configure_package_config_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/cmake/lcms2Config.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/lcms2Config.cmake
        INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/lcms2
    )

    write_basic_package_version_file(
        ${CMAKE_CURRENT_BINARY_DIR}/lcms2ConfigVersion.cmake
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
    )

    install(FILES
        ${CMAKE_CURRENT_BINARY_DIR}/lcms2Config.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/lcms2ConfigVersion.cmake
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/lcms2
    )
endif()
