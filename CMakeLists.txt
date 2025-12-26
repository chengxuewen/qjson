project(qjson)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DQT_NO_CAST_FROM_ASCII -DQT_NO_CAST_TO_ASCII")

# Force cmake 2.8.8 in order to have a decent support of Qt5
cmake_minimum_required(VERSION 2.8.8)
cmake_policy(SET CMP0003 NEW)

# Do not link against qtmain on Windows
if(POLICY CMP0020)
    cmake_policy(SET CMP0020 OLD)
endif()

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin" )
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib" )
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib" )

set(CMAKE_INSTALL_NAME_DIR ${LIB_INSTALL_DIR})

if("${CMAKE_BUILD_TYPE}" MATCHES "^Rel.*")
    add_definitions("-DQT_NO_DEBUG_OUTPUT")
endif("${CMAKE_BUILD_TYPE}" MATCHES "^Rel.*")

# Ability to disable verbose debug output
if(QJSON_VERBOSE_DEBUG_OUTPUT)
    add_definitions("-DQJSON_VERBOSE_DEBUG_OUTPUT")
endif(QJSON_VERBOSE_DEBUG_OUTPUT)

# On Windows debug library should have 'd' postfix.
if(WIN32)
    set(CMAKE_DEBUG_POSTFIX "d")
elseif(APPLE)
    set(CMAKE_DEBUG_POSTFIX "_debug")
endif(WIN32)

option(QJSON_BUILD_TESTS "Build tests" OFF)

# BUILD_SHARED_LIBS is cmake variable. Need to change default value.
option(BUILD_SHARED_LIBS "Build shared library" ON)

option(OSX_FRAMEWORK "Build a Mac OS X Framework")

# Allow to use shared Qt when compiling static version of QJSON.
# Has effect only when BUILD_SHARED_LIBS is OFF.
option(LINK_SHARED_QT "Force to link with shared Qt" OFF)
option(QT4_BUILD "Force building with Qt4 even if Qt5 is found" OFF)

set(FRAMEWORK_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/Library/Frameworks"
    CACHE PATH "Where to place qjson.framework if OSX_FRAMEWORK is selected")

find_package(Qt4 QUIET COMPONENTS QtCore QUIET)
if(NOT QT_FOUND)
    find_package(QT NAMES Qt6 Qt5 COMPONENTS Core QUIET)
    find_package(Qt${QT_VERSION_MAJOR} COMPONENTS Core QUIET)
endif()

if(QT_VERSION_MAJOR GREATER 4)
    message("Qt${QT_VERSION_MAJOR} found")

    INCLUDE_DIRECTORIES(${Qt${QT_VERSION_MAJOR}Core_INCLUDE_DIRS})
    ADD_DEFINITIONS(${Qt${QT_VERSION_MAJOR}Core_DEFINITIONS})
    set(PC_Requires "Qt${QT_VERSION_MAJOR}Core")
    set(QJSON_SUFFIX "-qt${QT_VERSION_MAJOR}")
    # Tell CMake to run moc when necessary:
    set(CMAKE_AUTOMOC ON)
    # As moc files are generated in the binary dir, tell CMake
    # to always look for includes there:
    set(CMAKE_INCLUDE_CURRENT_DIR ON)
    set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DQT_USE_QSTRINGBUILDER" )
    message("Enable QStringBuilder")
else()
    message("Qt5/Qt6 not found, searching for Qt4")

    # Don't use absolute path in qjson-targets-*.cmake
    # (This will have no effect with CMake < 2.8)
    # Workaround for no stdlib.h error. In this case it must be used with
    # -DQT_INCLUDE_DIRS_NO_SYSTEM=ON. So cmake must be invoked with
    # -DQT_USE_IMPORTED_TARGETS=OFF -DQT_INCLUDE_DIRS_NO_SYSTEM=ON
    # See https://bugzilla.redhat.com/show_bug.cgi?id=1470809 for details
    option(QT_USE_IMPORTED_TARGETS "Use imported targets" ON)

    # Find Qt4
    find_package(Qt4 4.5 REQUIRED QtCore)
    # QStringBuilder is supported since Qt 4.8 for both QString and QByteArray
    if(NOT (${QT_VERSION_MINOR} STRLESS "8"))
        message("Enable QStringBuilder")
        set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DQT_USE_QSTRINGBUILDER" )
    else()
        message("Disable QStringBuilder")
    endif()

    # Ensure to be linked with static Qt library on Widnows
    if(WIN32 AND NOT BUILD_SHARED_LIBS AND NOT LINK_SHARED_QT)
        string(REPLACE "-DQT_DLL" "" QT_DEFINITIONS "${QT_DEFINITIONS}")
        set(QT_DEFINITIONS ${QT_DEFINITIONS} "-DQT_NODLL")
    endif()

    # Include the cmake file needed to use qt4
    include(${QT_USE_FILE} )
    set(PC_Requires "QtCore")
endif()

if(NOT WIN32)
    set( QT_DONT_USE_QTGUI TRUE )
endif()


#add extra search paths for libraries and includes
set(LIB_SUFFIX "" CACHE STRING "Define suffix of directory name (32/64)" )
set(LIB_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/lib${LIB_SUFFIX}" CACHE STRING "Directory where lib will install")
set(INCLUDE_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE PATH "The directory the headers are installed in")
set(CMAKECONFIG_INSTALL_DIR "${LIB_INSTALL_DIR}/cmake/${CMAKE_PROJECT_NAME}${QJSON_SUFFIX}" CACHE PATH "Directory where to install QJSONConfig.cmake")

set(QJSON_LIB_MAJOR_VERSION "0")
set(QJSON_LIB_MINOR_VERSION "9")
set(QJSON_LIB_PATCH_VERSION "0")

set(QJSON_LIB_VERSION_STRING "${QJSON_LIB_MAJOR_VERSION}.${QJSON_LIB_MINOR_VERSION}.${QJSON_LIB_PATCH_VERSION}")

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib" )

# pkg-config
if(NOT WIN32)
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/QJson.pc.in
        ${CMAKE_CURRENT_BINARY_DIR}/QJson${QJSON_SUFFIX}.pc
        @ONLY)
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/QJson${QJSON_SUFFIX}.pc
        DESTINATION ${LIB_INSTALL_DIR}/pkgconfig)
endif(NOT WIN32)

# Subdirs
add_subdirectory(src)
if(KDE4_BUILD_TESTS OR QJSON_BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif(KDE4_BUILD_TESTS OR QJSON_BUILD_TESTS)

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
    IMMEDIATE @ONLY)

install(EXPORT qjson-export DESTINATION ${CMAKECONFIG_INSTALL_DIR} FILE QJSON${QJSON_SUFFIX}Targets.cmake)

# figure out the relative path from the installed Config.cmake file to the install prefix (which may be at
# runtime different from the chosen CMAKE_INSTALL_PREFIX if under Windows the package was installed anywhere)
# This relative path will be configured into the QJSONConfig.cmake
file(RELATIVE_PATH relInstallDir ${CMAKE_INSTALL_PREFIX}/${CMAKECONFIG_INSTALL_DIR} ${CMAKE_INSTALL_PREFIX} )

# cmake-modules
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/QJSONConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/QJSON${QJSON_SUFFIX}Config.cmake
    @ONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/QJSONConfigVersion.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/QJSON${QJSON_SUFFIX}ConfigVersion.cmake
    @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/QJSON${QJSON_SUFFIX}Config.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/QJSON${QJSON_SUFFIX}ConfigVersion.cmake
    DESTINATION "${CMAKECONFIG_INSTALL_DIR}")

add_custom_target(uninstall
    "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")
