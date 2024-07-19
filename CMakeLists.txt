cmake_minimum_required(VERSION 3.16...3.28)

project(
    LibTDMS
    VERSION 0.0.1
    DESCRIPTION "Check libusb event capability"
    LANGUAGES C CXX
)

#~ include(CheckIncludeFile)
include(CheckIPOSupported)
include(CheckPIESupported)
#~ include(CheckSymbolExists)
#~ include(CheckTypeSize)
include(CMakeDependentOption)
include(GNUInstallDirs)

find_program(CCACHE_PROGRAM ccache)
cmake_dependent_option(CCACHE_ENABLED
  "Use ccache to speed-up build." ON
  "CCACHE_PROGRAM" OFF
)
if(CCACHE_ENABLED)
  set(CMAKE_C_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
endif()

check_ipo_supported(RESULT IPO_SUPPORTED LANGUAGES C CXX)
check_pie_supported(OUTPUT_VARIABLE PIE_SUPPORTED LANGUAGES C CXX)

#~ configure_file(config.h.cmake config.h)

add_library(tdms_object OBJECT)
target_sources(tdms_object
    PRIVATE
        src/TDMS.c
)
set_target_properties(tdms_object
    PROPERTIES
        POSITION_INDEPENDENT_CODE ${PIE_SUPPORTED}
)
target_include_directories(tdms_object
    PUBLIC
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/include>
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/config>
      $<INSTALL_INTERFACE:${INCLUDEDIR}>
    PRIVATE
        config
        ${CMAKE_CURRENT_BINARY_DIR}
)

add_library(tdms_shared SHARED)
set_target_properties(tdms_shared
    PROPERTIES
        OUTPUT_NAME tdms
        INTERPROCEDURAL_OPTIMIZATION_RELEASE ${IPO_SUPPORTED}
        POSITION_INDEPENDENT_CODE ${PIE_SUPPORTED}
)
target_link_libraries(tdms_shared
    PUBLIC
        tdms_object
)

add_library(tdms_static STATIC $<TARGET_OBJECTS:tdms_object>)
set_target_properties(tdms_static
    PROPERTIES
        OUTPUT_NAME tdms
)
target_link_libraries(tdms_static
    PUBLIC
        tdms_object
)

add_library(LibTDMS::static ALIAS tdms_static)
add_library(LibTDMS::shared ALIAS tdms_shared)

if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
    install(TARGETS tdms_shared tdms_static tdms_object
        EXPORT LibTDMSTargets
        LIBRARY
            DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE
            DESTINATION ${CMAKE_INSTALL_LIBDIR}
            COMPONENT Development
        RUNTIME
            DESTINATION ${CMAKE_INSTALL_BINDIR}
    )
    install(
        FILES
            src/include/TDMS.h
            config/TDMS_config.h
        TYPE INCLUDE
        COMPONENT Development
    )

    include(CMakePackageConfigHelpers)
    write_basic_package_version_file(
        LibTDMSConfigVersion.cmake
        VERSION ${PACKAGE_VERSION}
        COMPATIBILITY AnyNewerVersion
    )

    export(EXPORT LibTDMSTargets
      FILE "${CMAKE_CURRENT_BINARY_DIR}/LibTDMSTargets.cmake"
      NAMESPACE LibTDMS::
    )
    configure_file(LibTDMSConfig.cmake.in
      "${CMAKE_CURRENT_BINARY_DIR}/LibTDMSConfig.cmake"
      COPYONLY
    )

    install(EXPORT LibTDMSTargets
      FILE
          LibTDMSTargets.cmake
      NAMESPACE
          LibTDMS::
      DESTINATION
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/LibTDMS
    )
    install(
        FILES
            "${CMAKE_CURRENT_BINARY_DIR}/LibTDMSConfig.cmake"
            "${CMAKE_CURRENT_BINARY_DIR}/LibTDMSConfigVersion.cmake"
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/LibTDMS
        COMPONENT Development
    )

    option(BUILD_EXAMPLES "Build example programs" OFF)
    if(BUILD_EXAMPLES)
        add_executable(libtdms_example_basic)
        target_sources(libtdms_example_basic
            PRIVATE
                example/basic/main.c
        )
        target_link_libraries(libtdms_example_basic
            PRIVATE
                tdms_static
        )
    endif()
endif()
