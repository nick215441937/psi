cmake_minimum_required(VERSION 3.1.0)
if(POLICY CMP0071)
	cmake_policy(SET CMP0071 OLD)
endif ()
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "")
endif()

#Psi or Psi+ detection
set( IS_PSIPLUS ON CACHE BOOL "Build Psi+ client")
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/src/widgets/psiwindowheader.cpp")
    message(STATUS "Psi+ features - FOUND")
    set(IS_PSIPLUS ON)
    project(psi-plus)
else()
    message(STATUS "Psi+ features - NOT FOUND")
    set(IS_PSIPLUS OFF)
    project(psi)
endif()

#Common options
option( BUNDLED_IRIS "Build iris library bundled" ON )
option( ONLY_PLUGINS "Build psi-plus plugins only" OFF )
option( USE_HUNSPELL "Build psi-plus with hunspell spellcheck" ON )
option( USE_ENCHANT "Build psi-plus with enchant spellcheck" OFF )
option( ENABLE_PLUGINS "Enable plugins" OFF )
option( ENABLE_WEBKIT "Enable webkit/webengine support" ON )
option( USE_WEBKIT "Use Webkit support instead of WebEngine" OFF )
option( USE_WEBENGINE "Use WebEngine support instead of Webkit" OFF )
option( USE_CCACHE "Use ccache utility if found" ON )
option( VERBOSE_PROGRAM_NAME "Verbose output binary name" OFF ) #Experimental
option( USE_CRASH "Enable builtin sigsegv handling" OFF )
option( USE_KEYCHAIN "Enable Qt5Keychain support" ON )
option( ONLY_BINARY "Build and install only binary file" OFF )
option( INSTALL_EXTRA_FILES "Install sounds, iconsets, certs, client_icons.txt, themes" ON )
#Iris options
option( USE_QJDNS "Use qjdns/jdns library. Disabled by default for Qt5" OFF )
option( SEPARATE_QJDNS "Build qjdns as separate lib" OFF )

if( IS_PSIPLUS )
    option( PRODUCTION "Build production version" OFF )
else()
    option( PRODUCTION "Build production version" ON )
endif()
#Windows or MXE only
option( USE_MXE "Use MXE cross-compilation" OFF )
option( ENABLE_PORTABLE "Create portable version of Psi+ in win32" OFF )
option( DEV_MODE "Enable prepare-bin-libs target for Windows OS" OFF )

if( USE_HUNSPELL AND USE_ENCHANT )
    message(FATAL_ERROR "Both flags USE_HUNSPELL and USE_ENCHANT cannot be enabled at the same time. Please disable one of them")
endif()

if( ONLY_BINARY AND INSTALL_EXTRA_FILES )
    message(FATAL_ERROR "Both flags ONLY_BINARY and INSTALL_EXTRA_FILES cannot be enabled at the same time. Please disable one of them")
endif()

set( QT4_BUILD OFF )

set( GLOBAL_DEPENDS_DEBUG_MODE ON )

set( CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH}" "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules" )

set( ENABLE_SQL OFF CACHE BOOL "Enable SQL history support")

if(ENABLE_WEBKIT)
    add_definitions(
        -DWEBKIT
    )
    find_package( Qt5Core REQUIRED )
    find_package( Qt5 COMPONENTS WebEngine QUIET )
    if(${Qt5Core_VERSION} VERSION_GREATER 5.6.0)
        if( Qt5WebEngine_FOUND AND (NOT USE_WEBKIT) )
            set( USE_WEBENGINE ON )
            add_definitions(
                -DWEBENGINE=1
            )
            message(STATUS "QtWebEngine - enabled")
        else()
            set( USE_WEBENGINE OFF )
        endif()
    endif()
    message(STATUS "Webkit - enabled")
endif()

if(USE_WEBENGINE AND USE_WEBKIT)
    message(FATAL_ERROR "Both flags USE_WEBENGINE and USE_WEBKIT cannot be enabled at the same time. Please disable one of them")
endif()

if(USE_WEBENGINE AND (NOT ENABLE_WEBKIT))
    message(FATAL_ERROR "USE_WEBENGINE flag is enabled but ENABLE_WEBKIT flag is disabled. Please disable USE_WEBENGINE or enable ENABLE_WEBKIT")
endif()

if(USE_WEBKIT AND (NOT ENABLE_WEBKIT))
    message(FATAL_ERROR "USE_WEBKIT flag is enabled but ENABLE_WEBKIT flag is disabled. Please disable USE_WEBKIT or enable ENABLE_WEBKIT")
endif()

if(EXISTS "${PROJECT_SOURCE_DIR}/src/historyimp.cpp")
    message(STATUS "SQL history - enabled")
    set(ENABLE_SQL ON)
else()
    set(ENABLE_SQL OFF)
endif()

if("${CMAKE_BUILD_TYPE}" STREQUAL "Debug" OR ("${CMAKE_BUILD_TYPE}" STREQUAL "RelWithDebInfo"))
    set(ISDEBUG ON)
endif()

if(ONLY_PLUGINS)
    set(ENABLE_PLUGINS ON)
endif()

if(USE_CRASH)
    add_definitions(-DUSE_CRASH)
endif()

#Detect MXE cross-compilation
if( (CMAKE_CROSSCOMPILING) AND (DEFINED MSYS) )
    message(STATUS "MXE environment detected")
    set(USE_MXE ON)
    message(STATUS "MXE root path: ${CMAKE_PREFIX_PATH}")
    # TODO: Check why this breaks build with GCC >= 6.0:
    #set(Qt5Keychain_DIR "${CMAKE_PREFIX_PATH}/lib/cmake/Qt5Keychain" CACHE STRING "Path to Qt5Keychain cmake files")
endif()


if(WIN32)
    if(MSVC)
        set(BUILD_ARCH "win64" CACHE STRING "CPU architecture win32/win64")
        set(SDK_PATH "e:/build/msvc2015-root" CACHE STRING "Path to SDK")
    else()
        set(BUILD_ARCH "i386" CACHE STRING "CPU architecture i386/x86_64")
        set(SDK_PATH "e:/build/psisdk" CACHE STRING "Path to SDK")
    endif()
    #autodetect 64bit architecture
    if(CMAKE_SIZEOF_VOID_P MATCHES "8")
        if(MSVC)
            set(BUILD_ARCH "win64")
        else()
            set(BUILD_ARCH "x86_64")
        endif()
        if(DEV_MODE)
            set(GST_SDK $ENV{GSTREAMER_1_0_ROOT_x86_64})
        endif()
    else()
        if(MSVC)
            set(BUILD_ARCH "win32")
        else()
            set(BUILD_ARCH "i386")
        endif()
        if(DEV_MODE)
            set(GST_SDK $ENV{GSTREAMER_1_0_ROOT_x86})
        endif()
    endif()

    message(STATUS "Detecting build architecture: ${BUILD_ARCH} detected")
    if(EXISTS "${SDK_PATH}")
        if(MSVC)
            set(QCA_DIR "${SDK_PATH}/" CACHE STRING "Path to QCA")
            set(IDN_ROOT "${SDK_PATH}/" CACHE STRING "Path to IDN library")
            set(HUNSPELL_ROOT "${SDK_PATH}/" CACHE STRING "Path to hunspell library")
            if(ENABLE_PLUGINS)
                set(LIBGCRYPT_ROOT "${SDK_PATH}/" CACHE STRING "Path to libgcrypt library")
                set(LIBGPGERROR_ROOT "${SDK_PATH}/" CACHE STRING "Path to libgpg-error library")
                set(LIBOTR_ROOT "${SDK_PATH}/" CACHE STRING "Path to libotr library")
                set(LIBTIDY_ROOT "${SDK_PATH}/" CACHE STRING "Path to libtidy library")
                set(SIGNAL_PROTOCOL_C_ROOT "${SDK_PATH}/" CACHE STRING "Path to libsignal-protocol-c library")
            endif()
            set(ZLIB_ROOT "${SDK_PATH}/" CACHE STRING "Path to zlib")
            set(OPENSSL_ROOT_DIR "${SDK_PATH}/" CACHE STRING "Path to openssl library")
            set(Qt5Keychain_DIR "${SDK_PATH}/lib/cmake/Qt5Keychain" CACHE STRING "Path to Qt5Keychain cmake files")
            #set(QJDNS_DIR "${SDK_PATH}/" CACHE STRING "Path to qjdns")
        else()
            set(QCA_DIR "${SDK_PATH}/qca/" CACHE STRING "Path to QCA")
            set(IDN_ROOT "${SDK_PATH}/libidn/" CACHE STRING "Path to IDN library")
            set(HUNSPELL_ROOT "${SDK_PATH}/hunspell/" CACHE STRING "Path to hunspell library")
            if(ENABLE_PLUGINS)
                set(LIBGCRYPT_ROOT "${SDK_PATH}/libgcrypt/" CACHE STRING "Path to libgcrypt library")
                set(LIBGPGERROR_ROOT "${SDK_PATH}/libgpg-error/" CACHE STRING "Path to libgpg-error library")
                set(LIBOTR_ROOT "${SDK_PATH}/libotr/" CACHE STRING "Path to libotr library")
                set(LIBTIDY_ROOT "${SDK_PATH}/libtidy/" CACHE STRING "Path to libtidy library")
                set(SIGNAL_PROTOCOL_C_ROOT "${SDK_PATH}/singal-protocol-c" CACHE STRING "Path to libsignal-protocol-c library")
            endif()
            set(ZLIB_ROOT "${SDK_PATH}/zlib/" CACHE STRING "Path to zlib")
            set(OPENSSL_ROOT_DIR "${SDK_PATH}/openssl" CACHE STRING "Path to openssl library")
            set(Qt5Keychain_DIR "${SDK_PATH}/qt5keychain/lib/cmake/Qt5Keychain" CACHE STRING "Path to Qt5Keychain cmake files")
            #set(QJDNS_DIR "${SDK_PATH}/qjdns/${SDK_SUFFIX}/" CACHE STRING "Path to qjdns")
            if(DEV_MODE)
                if(NOT GST_SDK)
                    set(GST_SDK "${SDK_PATH}/gstbundle/" CACHE STRING "Path to gstreamer SDK")
                endif()
                set(PSIMEDIA_DIR "${SDK_PATH}/psimedia/" CACHE STRING "Path to psimedia plugin")
            endif()
        endif()
    else()
        if(NOT USE_MXE)
            message(WARNING "Psi SDK not found at ${SDK_PATH}. Please set SDK_PATH variable or add Psi dependencies to PATH system environmet variable")
        endif()
    endif()
    set(PLUGINS_PATH "/plugins" CACHE STRING "Install suffix for plugins")
endif()

# Define LINUX on Linux like as WIN32 on Windows and APPLE on Mac OS X
if(UNIX AND NOT APPLE)
    set(LINUX ON)
    add_definitions(
        -DHAVE_X11
        -DHAVE_FREEDESKTOP
        -DAPP_PREFIX=${CMAKE_INSTALL_PREFIX}
        -DAPP_BIN_NAME=${PROJECT_NAME}
    )
endif()

# Qt dependencies make building very slow
# Track only .h files
include_regular_expression("^.*\\.h$")

# Put executable in build root dir
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/psi" )

if(APPLE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-overloaded-virtual")
else()
# Enable all warnings
    if(NOT MSVC)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -Wextra")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra")
        set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0")
        set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -O0")
    else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP")
        set(DEFAULT_DEBUG_FLAG "/ENTRY:mainCRTStartup /DEBUG /INCREMENTAL /SAFESEH:NO /MANIFEST:NO")
        set(DEFAULT_LINKER_FLAG "/ENTRY:mainCRTStartup /INCREMENTAL:NO /LTCG")
        set (CMAKE_EXE_LINKER_FLAGS_DEBUG        "${DEFAULT_DEBUG_FLAG}" CACHE STRING "" FORCE)
        set (CMAKE_EXE_LINKER_FLAGS_MINSIZEREL        "${DEFAULT_LINKER_FLAG}" CACHE STRING "" FORCE)
        set (CMAKE_EXE_LINKER_FLAGS_RELEASE        "${DEFAULT_LINKER_FLAG}" CACHE STRING "" FORCE)
        set (CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO    "${DEFAULT_DEBUG_FLAG}" CACHE STRING "" FORCE)
        set (CMAKE_MODULE_LINKER_FLAGS_DEBUG        "/DEBUG /INCREMENTAL /SAFESEH:NO /MANIFEST:NO" CACHE STRING "" FORCE)
        set (CMAKE_MODULE_LINKER_FLAGS_MINSIZEREL    "/INCREMENTAL:NO /LTCG" CACHE STRING "" FORCE)
        set (CMAKE_MODULE_LINKER_FLAGS_RELEASE        "/INCREMENTAL:NO /LTCG" CACHE STRING "" FORCE)
        set (CMAKE_MODULE_LINKER_FLAGS_RELWITHDEBINFO    "/DEBUG /INCREMENTAL:NO /MANIFEST:NO" CACHE STRING "" FORCE)
        set (CMAKE_SHARED_LINKER_FLAGS_DEBUG        "${DEFAULT_DEBUG_FLAG}" CACHE STRING "" FORCE)
        set (CMAKE_SHARED_LINKER_FLAGS_MINSIZEREL    "${DEFAULT_LINKER_FLAG}" CACHE STRING "" FORCE)
        set (CMAKE_SHARED_LINKER_FLAGS_RELEASE        "${DEFAULT_LINKER_FLAG}" CACHE STRING "" FORCE)
        set (CMAKE_SHARED_LINKER_FLAGS_RELWITHDEBINFO    "${DEFAULT_DEBUG_FLAG}" CACHE STRING "" FORCE)
        set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /ZI")
        set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /ZI /MTd")
        add_definitions(-DNOMINMAX)
        add_definitions(-D_CRT_SECURE_NO_WARNINGS)
        add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
        add_definitions(-D_CRT_NON_CONFORMING_SWPRINTFS)
        add_definitions(-D_SCL_SECURE_NO_WARNINGS)
        add_definitions(-D_WINSOCK_DEPRECATED_NO_WARNINGS)
        add_definitions(-D_UNICODE)
    endif()
endif()

if(ISDEBUG)
    message(STATUS "CXX debug flags: ${CMAKE_CXX_FLAGS_DEBUG}")
    message(STATUS "C debug flags: ${CMAKE_C_FLAGS_DEBUG}")
else()
    message(STATUS "CXX flags: ${CMAKE_CXX_FLAGS}")
    message(STATUS "C flags: ${CMAKE_C_FLAGS}")
endif()

if(NOT WIN32 AND ENABLE_PORTABLE)
    message(WARNING "Portable version creation possible only for Windows OS")
    set(ENABLE_PORTABLE OFF)
elseif(WIN32 AND ENABLE_PORTABLE)
    message(STATUS "Portable version - ENABLED")
endif()

if( PRODUCTION )
    message(STATUS "Production version - ENABLED")
endif()

if( DEV_MODE OR ENABLE_PORTABLE )
    message(STATUS "prepare-bin-libs target - ENABLED")
endif()

# Copy a list of files from one directory to another. Only full paths.
function(copy SOURCE DEST TARGET)
    if(EXISTS ${SOURCE})
        set(OUT_TARGET_FILE "${CMAKE_BINARY_DIR}/${TARGET}.cmake")

        string(REGEX REPLACE "\\\\+" "/" DEST "${DEST}")
        string(REGEX REPLACE "\\\\+" "/" SOURCE "${SOURCE}")

        if(NOT TARGET ${TARGET})
            file(REMOVE "${OUT_TARGET_FILE}")
            add_custom_target(${TARGET} COMMAND ${CMAKE_COMMAND} -P "${OUT_TARGET_FILE}")
        endif()

        if(IS_DIRECTORY ${SOURCE})
            # copy directory
            file(GLOB_RECURSE FILES "${SOURCE}/*")
            get_filename_component(SOURCE_DIR_NAME ${SOURCE} NAME)

            foreach(FILE ${FILES})
                file(RELATIVE_PATH REL_PATH ${SOURCE} ${FILE})
                set(REL_PATH "${SOURCE_DIR_NAME}/${REL_PATH}")
                get_filename_component(REL_PATH ${REL_PATH} DIRECTORY)
                set(DESTIN "${DEST}/${REL_PATH}")

                string(REGEX REPLACE "/+" "/" DESTIN ${DESTIN})
                string(REGEX REPLACE "/+" "/" FILE ${FILE})

                file(APPEND
                    "${OUT_TARGET_FILE}"
                    "file(INSTALL \"${FILE}\" DESTINATION \"${DESTIN}\" USE_SOURCE_PERMISSIONS)\n")
            endforeach()
        else()
            string(REPLACE "//" "/" DEST ${DEST})
            if(DEST MATCHES "/$")
                set(DIR "${DEST}")
                string(REGEX REPLACE "^(.+)/$" "\\1" DIR ${DIR})
            else()
                # need to copy and rename
                get_filename_component(DIR ${DEST} DIRECTORY)
                get_filename_component(FILENAME ${DEST} NAME)
                get_filename_component(SOURCE_FILENAME ${SOURCE} NAME)
            endif()
            file(APPEND
                "${OUT_TARGET_FILE}"
                "file(INSTALL \"${SOURCE}\" DESTINATION \"${DIR}\" USE_SOURCE_PERMISSIONS)\n")
            if(DEFINED FILENAME)
                file(APPEND
                    "${OUT_TARGET_FILE}"
                    "file(RENAME \"${DIR}/${SOURCE_FILENAME}\" \"${DIR}/${FILENAME}\")\n")
            endif()
        endif()
    endif()
endfunction()

if(USE_MXE)
    set(USE_CCACHE OFF)
endif()

if(USE_CCACHE)
    # Configure CCache if available
    find_program(CCACHE_PATH ccache DOC "Path to ccache")
    if(CCACHE_PATH)
        message(STATUS "Found ccache at ${CCACHE_PATH}")
        set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ${CCACHE_PATH})
        set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ${CCACHE_PATH})
    endif()
endif()


if(NOT ONLY_PLUGINS)
    if( USE_WEBENGINE )
        add_subdirectory( 3rdparty )
    endif()
    if( ${BUNDLED_IRIS} )
        add_subdirectory( iris )
    else( ${BUNDLED_IRIS} )
        find_package( Iris REQUIRED )
        include_directories(${Iris_INCLUDE_DIR})
    endif( ${BUNDLED_IRIS} )
    set( iris_LIB iris )
    add_subdirectory(src)
else()
    add_subdirectory(src/plugins)
endif()
