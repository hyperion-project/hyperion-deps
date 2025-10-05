cmake_minimum_required(VERSION 3.24.4)
project(dependencies)

set(HYPERION_DIR "" CACHE PATH "Path to cloned Hyperion.NG Repository")
if("${HYPERION_DIR}" STREQUAL "")
	message(FATAL_ERROR "Path to Hyperion.NG is missing! Add -DHYPERION_DIR=<PATH> to the cmake call.")
endif()

# Set C++ Standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Append the cmake module path
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# Include the helper macros
include(HelperMacros)

# Set build type
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
	set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Build type [Debug, RelWithDebInfo, Release, MinSizeRel]" FORCE)
	message(STATUS "No build type was specified, will default to ${CMAKE_BUILD_TYPE}")
endif()

# Set install location
set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}/install" CACHE PATH "Install prefix" FORCE)

# Build static/shared libraries
set(BUILD_SHARED_LIBS OFF CACHE BOOL "Build shared libraries")

# Search for Windows SDK
if(MSVC)
	find_package(WindowsSDK REQUIRED)
endif()

# Declare User-configurable options
set(PLATFORM_MATCHES_RPI OFF)
if(NOT DEFINED PLATFORM)
	if("${CMAKE_SYSTEM_PROCESSOR}" MATCHES "arm" OR "${CMAKE_SYSTEM_PROCESSOR}" MATCHES "aarch64")
		set(PLATFORM_MATCHES_RPI ON)
	endif()
else()
	if ("${PLATFORM}" MATCHES "rpi")
		set(PLATFORM_MATCHES_RPI ON)
	endif()
	message(STATUS "PLATFORM: ${PLATFORM}")
endif()

# Include CMake utility
include(ExternalProject)
include(FetchContent)

# macOS configuration
if(APPLE)
	set(APPLE_CONFIGURATION
		-DCMAKE_OSX_ARCHITECTURES:STRING=${CMAKE_OSX_ARCHITECTURES}
		-DCMAKE_OSX_SYSROOT:PATH=${CMAKE_OSX_SYSROOT}
		-DCMAKE_OSX_DEPLOYMENT_TARGET:STRING=${CMAKE_OSX_DEPLOYMENT_TARGET}
		-DCMAKE_MACOSX_RPATH:BOOL=${CMAKE_MACOSX_RPATH}
	)
endif()

#=============================================================================
# LibUSB (HIDAPI/LibFTDI need it)
#=============================================================================

ExternalProject_Add(libusb
	PREFIX				${CMAKE_CURRENT_BINARY_DIR}/_deps/libusb-cmake
	BUILD_ALWAYS		ON
	DOWNLOAD_COMMAND	""
	SOURCE_DIR			${HYPERION_DIR}/dependencies/external/libusb-cmake
	CMAKE_ARGS			-DBUILD_SHARED_LIBS:BOOL=OFF # Build libusb as static library
						-DLIBUSB_BUILD_TESTING:BOOL=OFF # Disable build of libusb tests
						-DLIBUSB_BUILD_EXAMPLES:BOOL=OFF # Disable build of libusb examples
						-DLIBUSB_INSTALL_TARGETS:BOOL=ON # Install libusb targets
						-DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
						-DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
	BUILD_BYPRODUCTS    <BINARY_DIR>/libusb-1.0${CMAKE_STATIC_LIBRARY_SUFFIX}
)

add_library(usb-1.0 STATIC IMPORTED GLOBAL)
add_dependencies(usb-1.0 libusb)
ExternalProject_Get_Property(libusb BINARY_DIR SOURCE_DIR)
set_target_properties(usb-1.0 PROPERTIES
	IMPORTED_LOCATION ${BINARY_DIR}/libusb-1.0${CMAKE_STATIC_LIBRARY_SUFFIX}
	INTERFACE_INCLUDE_DIRECTORIES ${SOURCE_DIR}/libusb/libusb
)

# =============================================================================
# LibFTDI
# =============================================================================

# Pass LibUSB include directory and library location to LibFTDI
get_target_property(LIBUSB_INCLUDE_DIR usb-1.0 INTERFACE_INCLUDE_DIRECTORIES)
set(LIBUSB_INCLUDE_DIR ${LIBUSB_INCLUDE_DIR} CACHE INTERNAL "")
get_target_property(LIBUSB_LIBRARIES usb-1.0 IMPORTED_LOCATION)
set(LIBUSB_LIBRARIES ${LIBUSB_LIBRARIES} CACHE INTERNAL "")

find_package(Git REQUIRED)
ExternalProject_Add(libftdi1
	DEPENDS				usb-1.0
	PREFIX				${CMAKE_CURRENT_BINARY_DIR}/libftdi1
	GIT_REPOSITORY		git://developer.intra2net.com/libftdi
	GIT_TAG				de9f01ece34d2fe6e842e0250a38f4b16eda2429
	PATCH_COMMAND		git reset --hard HEAD && git apply ${CMAKE_CURRENT_SOURCE_DIR}/patch/libftdi1.diff
	CMAKE_ARGS			-DSTATICLIBS:BOOL=ON # Build LibFTDI as static library
						-DEXAMPLES:BOOL=OFF #  Disable build of LibFTDI example programs
						-DFTDI_EEPROM:BOOL=OFF # Disable building ftdi_eeprom
						-DLIBUSB_INCLUDE_DIR:PATH=${LIBUSB_INCLUDE_DIR} # LibUSB include directory
						-DLIBUSB_LIBRARIES:PATH=${LIBUSB_LIBRARIES} # LibUSB library location
						-DCMAKE_INSTALL_LIBDIR:PATH=${CMAKE_INSTALL_PREFIX}/lib
						-DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
						-DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
						-DCMAKE_STATIC_LIBRARY_PREFIX_C:STRING=lib
						${APPLE_CONFIGURATION}
	BUILD_BYPRODUCTS    <BINARY_DIR>/src/libftdi1${CMAKE_STATIC_LIBRARY_SUFFIX}
)

#=============================================================================
# HIDAPI
#=============================================================================

cmake_path(SET HIDAPI ${HYPERION_DIR}/dependencies/external/hidapi)
FetchContent_Declare(hidapi SOURCE_DIR ${HIDAPI})

# Suppress warnings about "Compatibility with CMake < 3.5 will be removed from a future version of CMake"
set(CMAKE_WARN_DEPRECATED OFF CACHE INTERNAL "" FORCE)

# Use the libusb based implementation of HIDAPI (Linux only)
if(CMAKE_SYSTEM_NAME MATCHES "Linux")
	set(HIDAPI_WITH_HIDRAW OFF CACHE INTERNAL "Build HIDRAW-based implementation of HIDAPI")
endif()

# Install HIDAPI targets
set(HIDAPI_INSTALL_TARGETS ON CACHE INTERNAL "Enable the installation of HIDAPI targets")

# Disable build of HIDAPI tests
set(HIDAPI_BUILD_HIDTEST OFF CACHE INTERNAL "Disable small console test application hidtest")

FetchContent_MakeAvailable(hidapi)

#=============================================================================
# rpi_ws281x
#=============================================================================

if(PLATFORM_MATCHES_RPI)
	cmake_path(SET rpi_ws281x ${HYPERION_DIR}/dependencies/external/rpi_ws281x)
	FetchContent_Declare(ws281x SOURCE_DIR  ${rpi_ws281x})

	# Build rpi_ws281x as static library
	set(BUILD_SHARED ${BUILD_SHARED_LIBS} CACHE INTERNAL "Build statically version of rpi_ws281x")

	# Disable build of rpi_ws281x test application
	set(BUILD_TEST OFF CACHE INTERNAL "Build test application")

	FetchContent_MakeAvailable(ws281x)
endif()

#=============================================================================
# QMdnsEngine
#=============================================================================

# Allow to overwrite QT base directory
# Either supply QTDIR as -DQTDIR=<path> to cmake or set and environment variable QTDIR pointing to the Qt installation
# For Windows and macOS, the default Qt installation path are tried to resolved automatically
if(NOT DEFINED QTDIR)
	if(DEFINED ENV{QTDIR})
		set(QTDIR $ENV{QTDIR})
	else()
		if(MSVC)
			FIRSTSUBDIR(SUBDIRQT "C:/Qt")
			if(NOT ${SUBDIRQT} STREQUAL "")
				set(QTDIR "${SUBDIRQT}/msvc2019_64")
			endif()
		elseif (APPLE)
			foreach(QT_VERSION 6 5)
				execute_process(
					COMMAND brew --prefix qt@${QT_VERSION}
					RESULT_VARIABLE DETECT_QT
					OUTPUT_VARIABLE QT_LOCATION
					ERROR_QUIET
					OUTPUT_STRIP_TRAILING_WHITESPACE
				)
				if(${DETECT_QT} EQUAL 0 AND EXISTS ${QT_LOCATION})
					set(QTDIR ${QT_LOCATION})
					break()
				endif()
			endforeach()
		endif()
	endif()
endif()

if(DEFINED QTDIR)
	message(STATUS "Add QTDIR: ${QTDIR} to CMAKE_PREFIX_PATH")
	list(PREPEND CMAKE_PREFIX_PATH
		${QTDIR}
		${QTDIR}/lib
	)
endif()

# find Qt Network library for QMdnsEngine
find_package(QT NAMES Qt6 Qt5 COMPONENTS Network REQUIRED)
message(STATUS "Found Qt Version: ${QT_VERSION}")

if(${QT_VERSION_MAJOR} GREATER_EQUAL 6)
	set(QT_MIN_VERSION "6.2.2")
else()
	set(QT_MIN_VERSION "5.9.0")
endif()

if("${QT_VERSION}" VERSION_LESS "${QT_MIN_VERSION}")
	message(FATAL_ERROR "Your Qt version is to old! Minimum required ${QT_MIN_VERSION}")
endif()

find_package(Qt${QT_VERSION_MAJOR} ${QT_VERSION} COMPONENTS Network REQUIRED)
message(STATUS "Qt version used: ${QT_VERSION}")

cmake_path(SET QMdnsEngine ${HYPERION_DIR}/dependencies/external/qmdnsengine)
FetchContent_Declare(qmdns SOURCE_DIR ${QMdnsEngine})

# Suppress warnings about "Compatibility with CMake < 3.5 will be removed from a future version of CMake"
set(CMAKE_WARN_DEPRECATED OFF CACHE BOOL "" FORCE)

FetchContent_MakeAvailable(qmdns)

#=============================================================================
# FlatBuffers
#=============================================================================

cmake_path(SET FlatBuffers ${HYPERION_DIR}/dependencies/external/flatbuffers)
FetchContent_Declare(flatbuffers SOURCE_DIR ${FlatBuffers})

# Disable build of Flatbuffers tests
set(FLATBUFFERS_BUILD_TESTS OFF CACHE INTERNAL "Build Flatbuffers with tests")

FetchContent_MakeAvailable(flatbuffers)

# Suppress FlatBuffers warnings
if(CMAKE_C_COMPILER_ID MATCHES "GNU" AND TARGET flatbuffers)
	target_compile_options(flatbuffers PRIVATE -Wno-deprecated-declarations -Wno-psabi)
endif()

#=============================================================================
# Protocol Buffers
#=============================================================================

cmake_path(SET ProtocolBuffers ${HYPERION_DIR}/dependencies/external/protobuf)
FetchContent_Declare(protobuf
	SOURCE_DIR ${ProtocolBuffers}
	PATCH_COMMAND git apply ${CMAKE_CURRENT_SOURCE_DIR}/patch/protobuf.diff
 )

# Enable ProtoBuf verbose mode
set(protobuf_VERBOSE ON CACHE BOOL "Build protobuf verbose")

# Disable build of Protobuf tests
set(protobuf_BUILD_TESTS OFF CACHE INTERNAL "Build protobuf with tests")

# Disable build of Protobuf with zlib support
set(protobuf_WITH_ZLIB OFF CACHE INTERNAL "Build protobuf with zlib support")

# Disable build of libupb
set(protobuf_BUILD_LIBUPB OFF CACHE INTERNAL "Build libupb")

# Force all dependencies to be downloaded from GitHub
set(protobuf_FORCE_FETCH_DEPENDENCIES ON CACHE INTERNAL "Force all dependencies to be downloaded from GitHub")

# Build abseil (3rd party sub-module) with C++ version requirements
set(ABSL_PROPAGATE_CXX_STD ON CACHE INTERNAL "Build abseil-cpp with C++ version requirements propagated")

# Disable static linking of MSVC runtime libraries under Windows
if(WIN32)
	set(protobuf_MSVC_STATIC_RUNTIME OFF CACHE INTERNAL "Build protobuf static")
endif()

FetchContent_MakeAvailable(protobuf)

# Suppress Protobuf warnings
if(CMAKE_C_COMPILER_ID MATCHES "GNU" AND TARGET libprotobuf AND TARGET libprotoc AND TARGET absl)
	target_compile_options(libprotobuf PRIVATE -Wno-deprecated-declarations -Wno-psabi)
	target_compile_options(libprotoc PRIVATE -Wno-deprecated-declarations -Wno-psabi)
	target_compile_options(absl PRIVATE -Wno-deprecated-declarations -Wno-psabi)
endif()

#=============================================================================
# mbedTLS
#=============================================================================

cmake_path(SET mbedTLS ${HYPERION_DIR}/dependencies/external/mbedtls)
FetchContent_Declare(mbedtls SOURCE_DIR ${mbedTLS})

# Build mbedTLS as static library
set(USE_SHARED_MBEDTLS_LIBRARY ${BUILD_SHARED_LIBS} CACHE INTERNAL "Disable mbedTLS shared libraries")
set(USE_STATIC_MBEDTLS_LIBRARY !${BUILD_SHARED_LIBS} CACHE INTERNAL "Enable mbedTLS static libraries")

# Disable build of mbedTLS tests
set(ENABLE_TESTING OFF CACHE INTERNAL "Build mbedTLS tests")

# Disable fatal warnings
set(MBEDTLS_FATAL_WARNINGS OFF CACHE INTERNAL "Compiler warnings treated as errors")

# Disable build of mbedTLS program building
set(ENABLE_PROGRAMS OFF CACHE INTERNAL "Build mbedTLS programs")

# Enable export
set(DISABLE_PACKAGE_CONFIG_AND_INSTALL OFF CACHE INTERNAL "Disable package configuration, target export and installation")

FetchContent_MakeAvailable(mbedtls)

# Can be removed when the issue is resolved: https://github.com/Mbed-TLS/mbedtls/issues/9875
if(CMAKE_C_COMPILER_ID MATCHES "GNU" AND TARGET mbedcrypto)
	target_compile_options(mbedcrypto PRIVATE -fomit-frame-pointer)
endif()

#=============================================================================
# libjpeg-turbo (Windows)
#=============================================================================

if(MSVC)
	ExternalProject_Add(libjpeg-turbo
		PREFIX				${CMAKE_CURRENT_BINARY_DIR}/libjpeg-turbo
		GIT_REPOSITORY		https://github.com/libjpeg-turbo/libjpeg-turbo.git
		GIT_TAG				3.1.0
		CMAKE_ARGS			-DENABLE_SHARED:BOOL=${BUILD_SHARED_LIBS}
							-DENABLE_STATIC:BOOL=!${BUILD_SHARED_LIBS}
							-DCMAKE_INSTALL_LIBDIR:PATH=${CMAKE_INSTALL_PREFIX}/lib
							-DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
							-DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
	)
endif()
