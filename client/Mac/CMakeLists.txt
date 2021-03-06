
cmake_minimum_required (VERSION 2.8) 
project (MacFreeRDP) 
set(CMAKE_COLOR_MAKEFILE ON)

include(CheckIncludeFiles)
include(CheckLibraryExists)
include(CheckStructHasMember)
include(FindPkgConfig)
include(TestBigEndian)

# Include our extra modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/../../cmake/)

include(AutoVersioning)
include(ConfigOptions)
include(FindOptionalPackage)
include(CheckCCompilerFlag)
include(GNUInstallDirsWrapper)

# Default to debug build type
if(NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE "Debug")
endif()

# Default to build shared libs
if(NOT DEFINED BUILD_SHARED_LIBS)
    set(BUILD_SHARED_LIBS ON)
endif()

# Compiler-specific flags
if(CMAKE_COMPILER_IS_GNUCC)
	if(CMAKE_BUILD_TYPE STREQUAL "Release")
		set(CMAKE_C_FLAGS_RELEASE "-DNDEBUG")
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O2")
	endif()
	if(WITH_SSE2_TARGET)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -msse2")
	endif()
endif()

# Libraries that we have a hard dependency on
if(NOT DEFINED OPENSSL_INCLUDE_DIR OR NOT DEFINED OPENSSL_LIBRARIES)
    find_required_package(OpenSSL)
endif()

# Mac OS X
if(APPLE)
	# Set the include files for FreeRDP to the relative path
	set(FREERDP_INCLUDE_PATH ${CMAKE_SOURCE_DIR}/../../include/)
	set(FRAMEWORK_HEADERS_PATH /System/Library/Frameworks/Cocoa.framework/Versions/A/Headers/)
	include_directories (${FREERDP_INCLUDE_PATH} ${FRAMEWORK_HEADERS_PATH} /System/Library/Frameworks) 
	
	# set(CMAKE_OSX_SYSROOT MacOSX10.7.sdk) # uncomment to specify SDK version
	set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -mmacosx-version-min=10.4")
	set(GUI_TYPE MACOSX_BUNDLE)
	
	# Import libraries
	find_library(FOUNDATION_LIBRARY Foundation)
	message("+ Using foundation library ${FOUNDATION_LIBRARY}")
	find_library(COCOA_LIBRARY Cocoa)
	message("+ Using cocoa library ${COCOA_LIBRARY}")
	find_library(APPKIT_LIBRARY AppKit)
	message("+ Using appkit library ${APPKIT_LIBRARY}")
	
	message(" Current source dir: ${CMAKE_CURRENT_SOURCE_DIR}")
	# Set the OS X Bundle specific CMake variables which will be used to populate the plist for
	# the application bundle
	set(MACOSX_BUNDLE_INFO_STRING "${PROJECT_NAME}")
	set(MACOSX_BUNDLE_GUI_IDENTIFIER "com.freerdp.mac")
	set(MACOSX_BUNDLE_BUNDLE_IDENTIFIER "FreeRDP.Mac")
	set(MACOSX_BUNDLE_LONG_VERSION_STRING "${PROJECT_NAME} Version 1.0.1")
	set(MACOSX_BUNDLE_BUNDLE_NAME ${PROJECT_NAME})
	set(MACOSX_BUNDLE_SHORT_VERSION_STRING 1.0.1)
	set(MACOSX_BUNDLE_BUNDLE_VERSION 1.0.1)
	set(MACOSX_BUNDLE_COPYRIGHT "Copyright 2012. All Rights Reserved.")

	# Specific plist and NOT standard CMake variables
	set(MACOSX_BUNDLE_NSMAIN_NIB_FILE "MainMenu")
	set(MACOSX_BUNDLE_NSPRINCIPAL_CLASS "NSApplication")
	
	mark_as_advanced(COCOA_LIBRARY
						FOUNDATION_LIBRARY
						APPKIT_LIBRARY)
	set(EXTRA_LIBS ${COCOA_LIBRARY} ${FOUNDATION_LIBRARY} ${APPKIT_LIBRARY})
	set(APP_TYPE MACOSX_BUNDLE)
endif()

# OS X Interface Builder files
file (GLOB MacFreeRDP_XIBS
  	*.xib
)

# Headers
file (GLOB MacFreeRDP_Headers
	*.h
	)

# Source
file (GLOB MacFreeRDP_Source
	*.m 
	)

add_executable(MacFreeRDP 
	${APP_TYPE} 
	${MacFreeRDP_Headers}
	${MacFreeRDP_Source}
	${MacFreeRDP_XIBS})

# This is necessary for the xib file part below
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/Info.plist ${CMAKE_CURRENT_BINARY_DIR}/Info.plist)

# This allows for automatic xib to nib ibitool
set_target_properties(MacFreeRDP PROPERTIES RESOURCE "${MacFreeRDP_XIBS}")

# Automatic ref counting
set_target_properties(MacFreeRDP PROPERTIES XCODE_ATTRIBUTE_CLANG_ENABLE_OBJC_ARC YES)

# Support for automatic reference counting requires non-fragile abi.
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fobjc-nonfragile-abi")

# XCode project architecture to native architecture of build machine
# -----------------------------------------------------------------------------------------------------
# Issue: Had some issues with FreeRDP project building only 64 bit and
# MacFreeRDP attempting to link to both 32 and 64 for dual target.
# In the future the FreeRDP Xcode project should be pulled in for a couple of reasons:
# 1) better step-into debugging 2) automatic dependency compilation and multi-arch compilation + linkage
# If you know the solutions for 1 and 2, please add below.
set_target_properties(MacFreeRDP PROPERTIES XCODE_ATTRIBUTE_ARCHS "$(NATIVE_ARCH_ACTUAL)")

# Set the info plist to the custom instance
set_target_properties(MacFreeRDP PROPERTIES MACOSX_BUNDLE_INFO_PLIST ${CMAKE_CURRENT_BINARY_DIR}/Info.plist)

if(NOT WIN32)
	find_optional_package(MacAudio)
endif()

# Add all libraries
target_link_libraries(MacFreeRDP ${EXTRA_LIBS})
target_link_libraries(MacFreeRDP ${CMAKE_SOURCE_DIR}/../../xcode/libfreerdp-core/Debug/libfreerdp-core.dylib)
target_link_libraries(MacFreeRDP ${CMAKE_SOURCE_DIR}/../../xcode/libfreerdp-channels/Debug/libfreerdp-channels.dylib)
target_link_libraries(MacFreeRDP ${CMAKE_SOURCE_DIR}/../../xcode/libfreerdp-cache/Debug/libfreerdp-cache.dylib)
target_link_libraries(MacFreeRDP ${CMAKE_SOURCE_DIR}/../../xcode/libfreerdp-gdi/Debug/libfreerdp-gdi.dylib)
target_link_libraries(MacFreeRDP ${CMAKE_SOURCE_DIR}/../../xcode/libfreerdp-utils/Debug/libfreerdp-utils.dylib)
target_link_libraries(MacFreeRDP ${CMAKE_SOURCE_DIR}/../../xcode/libfreerdp-codec/Debug/libfreerdp-codec.dylib)
target_link_libraries(MacFreeRDP ${CMAKE_SOURCE_DIR}/../../xcode/libfreerdp-rail/Debug/libfreerdp-rail.dylib)
