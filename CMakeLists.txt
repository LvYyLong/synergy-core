# Synergy -- mouse and keyboard sharing utility
# Copyright (C) 2012-2016 Symless Ltd.
# Copyright (C) 2009 Nick Bolton
#
# This package is free software; you can redistribute it and/or
# modify it under the terms of the GNU General Public License
# found in the file LICENSE that should have accompanied this file.
#
# This package is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.

cmake_minimum_required (VERSION 3.4)
project (synergy-core C CXX)

if (DEFINED ENV{SYNERGY_NO_LEGACY})
    option (SYNERGY_BUILD_LEGACY_GUI "Build the legacy GUI" OFF)
    option (SYNERGY_BUILD_LEGACY_INSTALLER "Build the legacy installer" OFF)
else()
    option (SYNERGY_BUILD_LEGACY_GUI "Build the legacy GUI" ON)
    option (SYNERGY_BUILD_LEGACY_INSTALLER "Build the legacy installer" ON)
endif()

if (DEFINED ENV{SYNERGY_NO_TESTS})
    option (BUILD_TESTS "Override building of tests" OFF)
else()
    option (BUILD_TESTS "Override building of tests" ON)
    option (ENABLE_COVERAGE "Build with coverage")
endif()

if ($ENV{SYNERGY_ENTERPRISE})
    option (SYNERGY_ENTERPRISE "Build Enterprise" ON)
else()
    option (SYNERGY_ENTERPRISE "Build Enterprise" OFF)
endif()

set (CMAKE_CXX_STANDARD 14)
set (CMAKE_CXX_EXTENSIONS OFF)
set (CMAKE_CXX_STANDARD_REQUIRED ON)
set (CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
set (CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")

if (ENABLE_COVERAGE)
    # Add Code Coverage
    include(cmake/CodeCoverage.cmake)
    append_coverage_compiler_flags()
    setup_target_for_coverage_gcovr_xml(
            NAME coverage
            EXECUTABLE unittests
            BASE_DIRECTORY "${PROJECT_SOURCE_DIR}/src"
            EXCLUDE "ext/*")

endif()

if (NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_definitions (-DNDEBUG)
endif()

#
# Synergy version
#
include (cmake/Version.cmake)

# TODO: Find out why we need these, and remove them
if (COMMAND cmake_policy)
    cmake_policy (SET CMP0003 NEW)
    cmake_policy (SET CMP0005 NEW)
endif()

# Add headers to source list
if (${CMAKE_GENERATOR} STREQUAL "Unix Makefiles")
    set (SYNERGY_ADD_HEADERS FALSE)
else()
    set (SYNERGY_ADD_HEADERS TRUE)
endif()



set (libs)
include_directories (BEFORE SYSTEM ${PROJECT_SOURCE_DIR}/ext/googletest/googletest/include)

if (UNIX)
    if (NOT APPLE)
        set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")
    endif()

    # For config.h, detect the libraries, functions, etc.
    include (CheckIncludeFiles)
    include (CheckLibraryExists)
    include (CheckFunctionExists)
    include (CheckTypeSize)
    include (CheckIncludeFileCXX)
    include (CheckSymbolExists)
    include (CheckCSourceCompiles)

    check_include_file_cxx (istream HAVE_ISTREAM)
    check_include_file_cxx (ostream HAVE_OSTREAM)
    check_include_file_cxx (sstream HAVE_SSTREAM)

    check_include_files (inttypes.h HAVE_INTTYPES_H)
    check_include_files (locale.h HAVE_LOCALE_H)
    check_include_files (memory.h HAVE_MEMORY_H)
    check_include_files (stdlib.h HAVE_STDLIB_H)
    check_include_files (strings.h HAVE_STRINGS_H)
    check_include_files (string.h HAVE_STRING_H)
    check_include_files (sys/select.h HAVE_SYS_SELECT_H)
    check_include_files (sys/socket.h HAVE_SYS_SOCKET_H)
    check_include_files (sys/stat.h HAVE_SYS_STAT_H)
    check_include_files (sys/time.h HAVE_SYS_TIME_H)
    check_include_files (sys/utsname.h HAVE_SYS_UTSNAME_H)
    check_include_files (unistd.h HAVE_UNISTD_H)
    check_include_files (wchar.h HAVE_WCHAR_H)

    check_function_exists (getpwuid_r HAVE_GETPWUID_R)
    check_function_exists (gmtime_r HAVE_GMTIME_R)
    check_function_exists (nanosleep HAVE_NANOSLEEP)
    check_function_exists (poll HAVE_POLL)
    check_function_exists (sigwait HAVE_POSIX_SIGWAIT)
    check_function_exists (strftime HAVE_STRFTIME)
    check_function_exists (vsnprintf HAVE_VSNPRINTF)
    check_function_exists (inet_aton HAVE_INET_ATON)

    # For some reason, the check_function_exists macro doesn't detect
    # the inet_aton on some pure Unix platforms (e.g. sunos5). So we
    # need to do a more detailed check and also include some extra libs.
    if (NOT HAVE_INET_ATON)
        set (CMAKE_REQUIRED_LIBRARIES nsl)

        check_c_source_compiles (
            "#include <arpa/inet.h>\n int main() { inet_aton (0, 0); }"
            HAVE_INET_ATON_ADV)

        set (CMAKE_REQUIRED_LIBRARIES)

        if (HAVE_INET_ATON_ADV)
            # Override the previous fail.
            set (HAVE_INET_ATON 1)

            # Assume that both nsl and socket will be needed,
            # it seems safe to add socket on the back of nsl,
            # since socket only ever needed when nsl is needed.
            list (APPEND libs nsl socket)
        endif()

    endif()

    check_type_size (char SIZEOF_CHAR)
    check_type_size (int SIZEOF_INT)
    check_type_size (long SIZEOF_LONG)
    check_type_size (short SIZEOF_SHORT)

    # pthread is used on both Linux and Mac
    check_library_exists ("pthread" pthread_create "" HAVE_PTHREAD)
    if (HAVE_PTHREAD)
        list (APPEND libs pthread)
    else()
        message (FATAL_ERROR "Missing library: pthread")
    endif()


    if (APPLE)
        set (CMAKE_CXX_FLAGS "--sysroot ${CMAKE_OSX_SYSROOT} ${CMAKE_CXX_FLAGS} -DGTEST_USE_OWN_TR1_TUPLE=1")

        if(NOT CMAKE_OSX_DEPLOYMENT_TARGET)
            set(CMAKE_OSX_DEPLOYMENT_TARGET 11.0)
        endif()

        if(CMAKE_OSX_DEPLOYMENT_TARGET GREATER_EQUAL 11.0)
            set(SYNERGY_OSX_DEPLOYMENT_TARGET 1100)
        elseif(CMAKE_OSX_DEPLOYMENT_TARGET GREATER_EQUAL 10.15)
            set(SYNERGY_OSX_DEPLOYMENT_TARGET 1015)
        elseif(CMAKE_OSX_DEPLOYMENT_TARGET GREATER_EQUAL 10.14)
            set(SYNERGY_OSX_DEPLOYMENT_TARGET 1014)
        else()
            set(SYNERGY_OSX_DEPLOYMENT_TARGET 1013)
        endif()
        add_compile_definitions(OSX_DEPLOYMENT_TARGET=${SYNERGY_OSX_DEPLOYMENT_TARGET})

        find_library (lib_ScreenSaver ScreenSaver)
        find_library (lib_IOKit IOKit)
        find_library (lib_ApplicationServices ApplicationServices)
        find_library (lib_Foundation Foundation)
        find_library (lib_Carbon Carbon)

        list (APPEND libs
            ${lib_ScreenSaver}
            ${lib_IOKit}
            ${lib_ApplicationServices}
            ${lib_Foundation}
            ${lib_Carbon}
        )

        if(SYNERGY_OSX_DEPLOYMENT_TARGET GREATER_EQUAL 1014)
            find_library (lib_UserNotifications UserNotifications)
            list (APPEND libs
                ${lib_UserNotifications}
            )
        endif()

    else() # not-apple

        # add include dir for bsd (posix uses /usr/include/)
        set (CMAKE_INCLUDE_PATH "${CMAKE_INCLUDE_PATH}:/usr/local/include")

        set (XKBlib "X11/Xlib.h;X11/XKBlib.h")
        set (CMAKE_EXTRA_INCLUDE_FILES "${XKBlib};X11/extensions/Xrandr.h")
        check_type_size ("XRRNotifyEvent" X11_EXTENSIONS_XRANDR_H)
        set (HAVE_X11_EXTENSIONS_XRANDR_H "${X11_EXTENSIONS_XRANDR_H}")
        set (CMAKE_EXTRA_INCLUDE_FILES)

        check_include_files ("${XKBlib};X11/extensions/dpms.h" HAVE_X11_EXTENSIONS_DPMS_H)
        check_include_files ("X11/extensions/Xinerama.h" HAVE_X11_EXTENSIONS_XINERAMA_H)
        check_include_files ("${XKBlib};X11/extensions/XKBstr.h" HAVE_X11_EXTENSIONS_XKBSTR_H)
        check_include_files ("X11/extensions/XKB.h" HAVE_XKB_EXTENSION)
        check_include_files ("X11/extensions/XTest.h" HAVE_X11_EXTENSIONS_XTEST_H)
        check_include_files ("${XKBlib}" HAVE_X11_XKBLIB_H)
        check_include_files ("X11/extensions/XInput2.h" HAVE_XI2)

        if (HAVE_X11_EXTENSIONS_DPMS_H)
            # Assume that function prototypes declared, when include exists.
            set (HAVE_DPMS_PROTOTYPES 1)
        endif()

        if (NOT HAVE_X11_XKBLIB_H)
            message (FATAL_ERROR "Missing header: " ${XKBlib})
        endif()

        check_library_exists ("SM;ICE" IceConnectionNumber "" HAVE_ICE)
        check_library_exists ("Xext;X11" DPMSQueryExtension "" HAVE_Xext)
        check_library_exists ("Xtst;Xext;X11" XTestQueryExtension "" HAVE_Xtst)
        check_library_exists ("Xinerama" XineramaQueryExtension "" HAVE_Xinerama)
        check_library_exists ("Xi" XISelectEvents "" HAVE_Xi)
        check_library_exists ("Xrandr" XRRQueryExtension "" HAVE_Xrandr)

        if (HAVE_ICE)

            # Assume we have SM if we have ICE.
            set (HAVE_SM 1)
            list (APPEND libs SM ICE)

        endif()

        if (!X11_xkbfile_FOUND)
            message (FATAL_ERROR "Missing library: xkbfile")
        endif()

        if (HAVE_Xtst)

            # Xtxt depends on X11.
            set (HAVE_X11)
            list (APPEND libs Xtst X11 xkbfile)

        else()

            message (FATAL_ERROR "Missing library: Xtst")

        endif()

        if (HAVE_Xext)
            list (APPEND libs Xext)
        endif()

        if (HAVE_Xinerama)
            list (APPEND libs Xinerama)
        else (HAVE_Xinerama)
            if (HAVE_X11_EXTENSIONS_XINERAMA_H)
                set (HAVE_X11_EXTENSIONS_XINERAMA_H 0)
                message (WARNING "Old Xinerama implementation detected, disabled")
            endif()
        endif()

        if (HAVE_Xrandr)
            list (APPEND libs Xrandr)
        endif()

        # this was outside of the linux scope,
        # not sure why, moving it back inside.
        if (HAVE_Xi)
            list (APPEND libs Xi)
        endif()

    endif()

    # For config.h, set some static values; it may be a good idea to make
    # these values dynamic for non-standard UNIX compilers.
    set (ACCEPT_TYPE_ARG3 socklen_t)
    set (HAVE_CXX_BOOL 1)
    set (HAVE_CXX_CASTS 1)
    set (HAVE_CXX_EXCEPTIONS 1)
    set (HAVE_CXX_MUTABLE 1)
    set (HAVE_CXX_STDLIB 1)
    set (HAVE_PTHREAD_SIGNAL 1)
    set (SELECT_TYPE_ARG1 int)
    set (SELECT_TYPE_ARG234 " (fd_set *)")
    set (SELECT_TYPE_ARG5 " (struct timeval *)")
    set (STDC_HEADERS 1)
    set (TIME_WITH_SYS_TIME 1)
    set (HAVE_SOCKLEN_T 1)

    # For config.h, save the results based on a template (config.h.in).
    configure_file (res/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/src/lib/config.h)

    add_definitions (-DSYSAPI_UNIX=1 -DHAVE_CONFIG_H)

    if (APPLE)
        add_definitions (-DWINAPI_CARBON=1 -D_THREAD_SAFE)
    else()
        add_definitions (-DWINAPI_XWINDOWS=1)
    endif()

elseif (${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP /D _BIND_TO_CURRENT_VCLIBS_VERSION=1")
    set (CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MD /O2 /Ob2")

    list (APPEND libs Wtsapi32 Userenv Wininet comsuppw Shlwapi)

    add_definitions (
        /DWIN32
        /D_WINDOWS
        /D_CRT_SECURE_NO_WARNINGS
        /DSYNERGY_VERSION=\"${SYNERGY_VERSION}\"
        /D_XKEYCHECK_H
    )
endif()

#
# OpenSSL
#
# Apple has to use static libraries because
# "Use of the Apple-provided OpenSSL libraries by apps is strongly discouraged."
# https://developer.apple.com/library/archive/documentation/Security/Conceptual/cryptoservices/SecureNetworkCommunicationAPIs/SecureNetworkCommunicationAPIs.html
if(APPLE OR DEFINED ENV{SYNERGY_STATIC_OPENSSL})
    set(OPENSSL_USE_STATIC_LIBS TRUE)
endif()
find_package(OpenSSL REQUIRED)

#
# Check submodules
#
find_package(Git QUIET)
if(GIT_FOUND AND EXISTS "${PROJECT_SOURCE_DIR}/.git")
# Update submodules as needed
    option(GIT_SUBMODULE "Check submodules during build" ON)
    if(GIT_SUBMODULE)
        message(STATUS "Submodule update")
        execute_process(COMMAND ${GIT_EXECUTABLE} submodule update --init --recursive
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        RESULT_VARIABLE GIT_SUBMOD_RESULT)
        if(NOT GIT_SUBMOD_RESULT EQUAL "0")
            message(FATAL_ERROR "git submodule update --init failed with ${GIT_SUBMOD_RESULT}, please checkout submodules")
        endif()
    endif()
endif()

#
# Google Test
#
if(BUILD_TESTS AND NOT EXISTS "${PROJECT_SOURCE_DIR}/ext/googletest/CMakeLists.txt")
    message(FATAL_ERROR "The submodules were not downloaded! GIT_SUBMODULE was turned off or failed. Please update submodules and try again.")
endif()


#
# Configure_file... but for directories, recursively.
#
macro (configure_files srcDir destDir)
    message (STATUS "Configuring directory ${destDir}")
    make_directory (${destDir})

    file (GLOB_RECURSE sourceFiles RELATIVE ${srcDir} ${srcDir}/*)
    file (GLOB_RECURSE templateFiles LIST_DIRECTORIES false RELATIVE ${srcDir} ${srcDir}/*.in)
    list (REMOVE_ITEM sourceFiles ${templateFiles})

    foreach (sourceFile ${sourceFiles})
        set (sourceFilePath ${srcDir}/${sourceFile})
        if (IS_DIRECTORY ${sourceFilePath})
            message (STATUS "Copying directory ${sourceFile}")
            make_directory (${destDir}/${sourceFile})
        else()
            message (STATUS "Copying file ${sourceFile}")
            configure_file (${sourceFilePath} ${destDir}/${sourceFile} COPYONLY)
        endif()
    endforeach (sourceFile)

    foreach (templateFile ${templateFiles})
        set (sourceTemplateFilePath ${srcDir}/${templateFile})
                string (REGEX REPLACE "\.in$" "" templateFile ${templateFile})
        message (STATUS "Configuring file ${templateFile}")
        configure_file (${sourceTemplateFilePath} ${destDir}/${templateFile} @ONLY)
    endforeach (templateFile)
endmacro (configure_files)

macro(generate_versionfile)
    if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin" OR ${CMAKE_SYSTEM_NAME} MATCHES "Linux|.*BSD|DragonFly")
        FILE(WRITE ${CMAKE_BINARY_DIR}/version
        "export SYNERGY_VERSION_MAJOR=\"${SYNERGY_VERSION_MAJOR}\"\n"
        "export SYNERGY_VERSION_MINOR=\"${SYNERGY_VERSION_MINOR}\"\n"
        "export SYNERGY_VERSION_PATCH=\"${SYNERGY_VERSION_PATCH}\"\n"
        "export SYNERGY_VERSION_BUILD=\"${SYNERGY_VERSION_BUILD}\"\n"
        "export SYNERGY_VERSION_STAGE=\"${SYNERGY_VERSION_STAGE}\"\n")
    elseif(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
        FILE(WRITE ${CMAKE_BINARY_DIR}/version.bat
        "SET SYNERGY_VERSION_MAJOR=${SYNERGY_VERSION_MAJOR}\n"
        "SET SYNERGY_VERSION_MINOR=${SYNERGY_VERSION_MINOR}\n"
        "SET SYNERGY_VERSION_PATCH=${SYNERGY_VERSION_PATCH}\n"
        "SET SYNERGY_VERSION_BUILD=${SYNERGY_VERSION_BUILD}\n"
        "SET SYNERGY_VERSION_STAGE=${SYNERGY_VERSION_STAGE}\n")
    endif()
endmacro(generate_versionfile)

generate_versionfile()

if (${SYNERGY_BUILD_LEGACY_INSTALLER})
#
# macOS app Bundle
#
if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set (CMAKE_INSTALL_RPATH "@loader_path/../Libraries;@loader_path/../Frameworks")
    set (SYNERGY_BUNDLE_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/dist/macos/bundle)
    set (SYNERGY_BUNDLE_DIR ${CMAKE_BINARY_DIR}/bundle)
    set (SYNERGY_BUNDLE_APP_DIR ${SYNERGY_BUNDLE_DIR}/Synergy.app)
    set (SYNERGY_BUNDLE_BINARY_DIR ${SYNERGY_BUNDLE_APP_DIR}/Contents/MacOS)
    configure_files (${SYNERGY_BUNDLE_SOURCE_DIR} ${SYNERGY_BUNDLE_DIR})
endif()

#
# Windows installer
#
if (${CMAKE_SYSTEM_NAME} MATCHES "Windows")
        message (STATUS "Configuring the v1 installer")
        set(QT_PATH $ENV{CMAKE_PREFIX_PATH})
        configure_files (${CMAKE_CURRENT_SOURCE_DIR}/dist/wix ${CMAKE_BINARY_DIR}/installer)
endif()

#
# Linux installation
#
if (${CMAKE_SYSTEM_NAME} MATCHES "Linux|.*BSD|DragonFly")
    configure_files (${CMAKE_CURRENT_SOURCE_DIR}/dist/rpm ${CMAKE_BINARY_DIR}/rpm)
    install(FILES res/synergy.svg DESTINATION share/icons/hicolor/scalable/apps)
    if("${VERSION_MAJOR}" STREQUAL "2")
        install(FILES res/synergy2.desktop DESTINATION share/applications)
    else()
        install(FILES res/synergy.desktop DESTINATION share/applications)
    endif()
endif()

else()
    message (STATUS "NOT configuring the v1 installer")
endif()
add_subdirectory (src)
