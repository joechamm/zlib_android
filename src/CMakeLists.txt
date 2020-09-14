cmake_minimum_version(VERSION 3.15)

project(zlib_android C)

set(VERSION "1.2.11")

option(ASM686 "Enable building i686 assembly implementation")
option(AMD64 "Enable building amd64 assembly implementation")

include(CheckTypeSize)
include(CheckFunctionExists)
include(CheckIncludeFile)
include(CheckCSourceCompiles)
enable_testing()

check_include_file(sys/types.h HAVE_SYS_TYPES_H)
check_include_file(stdint.h HAVE_STDINT_H)
check_include_file(stddef.h HAVE_STDDEF_H)

set(CMAKE_REQUIRED_DEFINITIONS -D_LARGEFILE64_SOURCE=1)
if(HAVE_SYS_TYPES_H)
	list(APPEND CMAKE_REQUIRED_DEFINITIONS -DHAVE_SYS_TYPES_H)
endif()
if(HAVE_STDINT_H)
	list(APPEND CMAKE_REQUIRED_DEFINITIONS -DHAVE_STDINT_H)
endif()
if(HAVE_STDDEF_H)
	list(APPEND CMAKE_REQUIRED_DEFINITIONS -DHAVE_STDDEF_H)
endif()
check_type_size(off64_t OFF64_T)
if(HAVE_OFF64_T)
	add_definitions(-D_LARGEFILE64_SOURCE=1)
endif()
set(CMAKE_REQUIRED_DEFINITIONS)

# check for fseeko
check_function_exists(fseeko HAVE_FSEEKO)
if(NOT HAVE_FSEEKO)
	add_definitions(-DNO_FSEEKO)
endif()

# check for unistd.h
check_include_file(unistd.h Z_HAVE_UNISTD_H)

if(MSVC)
	set(CMAKE_DEBUG_POSTFIX "d")
	add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
	add_definitions(-D_CRT_NONSTDC_NO_DEPRECATE)
	include_directories(${CMAKE_CURRENT_SOURCE_DIR})
endif()

add_library(zlib_android_compile_flags INTERFACE)
target_compile_features(zlib_android_compile_flags INTERFACE c_std_11)

# add compiler warning flags just when building this project via the BUILD_INTERFACE genex
set(gcc_like_c "$<COMPILE_LANG_AND_ID:C,ARMCC,ARMClang,Clang,GNU>")
set(msvc_c "$<COMPILE_LANG_AND_ID:C,MSVC>")

target_compile_options(zlib_android_compile_flags INTERFACE
	"$<${gcc_like_cxx}:$<BUILD_INTERFACE:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>>"
	"$<${msvc_cxx}:$<BUILD_INTERFACE:-W3>>"
	)

set(CMAKE_SYSTEM_NAME Android)
set(CMAKE_ANDROID_NDK /home/joechamm/snap/androidsdk/current/AndroidSDK/ndk-bundle)

option(CMAKE_SYSTEM_VERSION "Android Platform Version (valid range 16-30)" 21) # API level
option(CMAKE_ANDROID_ARCH_ABI "Android ABI (armeabi-v7a, arm64-v8a, x86, x86_64)" arm64-v8a)
option(CMAKE_ANDROID_STL_TYPE "Android STL Type (gnustl_static, gnustl_shared, c++_static, c++_shared)" gnustl_static)
option(BUILD_SHARED_LIBS "Shared Libraries" ON)

set(INSTALL_BIN_DIR "${CMAKE_INSTALL_PREFIX}/${CMAKE_ANDROID_ARCH_ABI}/bin" CACHE PATH "Installation directory for executables")
set(INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/${CMAKE_ANDROID_ARCH_ABI}/lib" CACHE PATH "Installation directory for libraries")
set(INSTALL_INC_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE PATH "Installation directory for headers")
set(INSTALL_MAN_DIR "${CMAKE_INSTALL_PREFIX}/share/man" CACHE PATH "Installation for manual pages")
set(INSTALL_PKGCONFIG_DIR "${CMAKE_INSTALL_PREFIX}/share/pkgconfig" CACHE PATH "Installation for pkgconfig (.pc) files")

# set staging prefix for cross compile
set(CMAKE_STAGING_PREFIX "${CMAKE_INSTALL_PREFIX}/${CMAKE_ANDROID_ARCH_ABI}/staging")

if(NOT CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR)
	# if we're doing an out of source build and the user has a zconf.h in their source tree
	if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h)
		message(STATUS "Renaming")
		message(STATUS "	${CMAKE_CURRENT_SOURCE_DIR}/zconf.h")
		message(STATUS "to 'zconf.h.included" because this file is included with zlib")
		message(STATUS "but Cmake generates it automatically in the build directory.")
		file(RENAME ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h.included)
	endif()
endif()

set(ZLIB_ANDROID_PC ${CMAKE_CURRENT_BINARY_DIR}/zlib.pc)
configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/zlib.pc.cmakein
	${ZLIB_ANDROID_PC} @ONLY)
configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h.cmakein
	${CMAKE_CURRENT_BINARY_DIR}/zconf.h @ONLY)
include_directories(${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_SOURCE_DIR})

set(ZLIB_ANDROID_PUBLIC_HEADERS
	${CMAKE_CURRENT_BINARY_DIR}/zconf.h
	zlib.h
	)

set(ZLIB_ANDROID_PRIVATE_HEADERS
	crc32.h
	deflate.h
	gzguts.h
	inffast.h
	inffixed.h
	inflate.h
	inftrees.h
	trees.h
	zutil.h
	)

set(ZLIB_ANDROID_SRCS
	adler32.c
	compress.c
	crc32.c
	deflate.c
	gzclose.c
	gzlib.c
	gzread.c
	gzwrite.c
	inflate.c
	infback.c
	inftrees.c
	inffast.c
	trees.c
	uncompr.c
	zutil.c
	)


if(NOT MINGW)
	set(ZLIB_DLL_SRCS
		win32/zlib1.rc # if present will override custom build rule below
		)
endif()

if(CMAKE_COMPILER_IS_GNUCC)
	if(ASM686)
		set(ZLIB_ASMS contrib/asm686/match.S)
	elseif(AMD64)
		set(ZLIB_ASMS contrib/amd64/amd64-match.S)
	endif()

	if(ZLIB_ASMS)
		add_definitions(-DASMV)
		set_source_file_properties(${ZLIB_ASMS} PROPERTIES LANGUAGE C COMPILE_FLAGS -DNO_UNDERLINE)
	endif()
endif()

if(MSVC)
	if(ASM686)
		ENABLE_LANGUAGE(ASM_MASM)
		set(ZLIB_ASMS 
			contrib/masmx86/inffas32.asm
			contrib/masmx86/match686.asm
			)
	elseif(AMD64)
		ENABLE_LANGUAGE(ASM_MASM)
		set(ZLIB_ASMS
			contrib/masmx64/gvmat64.asm
			contrib/masmx64/inffasx64.asm
			)
	endif()
	if(ZLIB_ASMS)
		add_definitions(-DASMV -DASMINF)
	endif()
endif()


file(READ ${CMAKE_CURRENT_SOURCE_DIR}/zlib.h _zlib_h_contents)
string(REGEX REPLACE ".*#define[ \t]+ZLIB_VERSION[ \t]+\"([-0-9A-Za-z.]+)\".*"
	"\\1" ZLIB_ANDROID_FULL_VERSION ${_zlib_h_contents})

if(MINGW)
	# This gets us DLL resource information when compiling on MinGW
	if(NOT CMAKE_RC_COMPILER)
		set(CMAKE_RC_COMPILER windres.exe)
	endif()

	add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj
		COMMAND ${CMAKE_RC_COMPILER}
		-D GCC_WINDRES
		-I ${CMAKE_CURRENT_SOURCE_DIR}
		-I ${CMAKE_CURRENT_BINARY_DIR}
		-o ${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj
		-i ${CMAKE_CURRENT_SOURCE_DIR}/win32/zlib1.rc)
	set(ZLIB_DLL_SRCS ${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj)
endif(MINGW)

if(NOT BUILD_SHARED_LIBS)
	unset(ZLIB_DLL_SRCS)
endif()

add_library(zlib ${ZLIB_ANDROID_SRCS} ${ZLIB_ASMS} ${ZLIB_DLL_SRCS} ${ZLIB_ANDROID_PUBLIC_HEADERS} ${ZLIB_ANDROID_PRIVATE_HEADERS})
if(BUILD_SHARED_LIBS)
	set_target_properties(zlib PROPERTIES DEFINE_SYMBOL ZLIB_DLL)
	set_target_properties(zlib PROPERTIES SOVERSION 1)
endif()

set_target_properties(zlib PROPERTIES VERSION ${ZLIB_ANDROID_FULL_VERSION})
set_target_properties(zlib PROPERTIES OUTPUT_NAME z)
set_target_properties(zlib PROPERTIES LINK_FLAGS "-Wl,--version-script,\"${CMAKE_CURRENT_SOURCE_DIR}/zlib.map\"")

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
	install(TARGETS zlib
		RUNTIME DESTINATION "${INSTALL_BIN_DIR}"
		ARCHIVE DESTINATION "${INSTALL_LIB_DIR}"
		LIBRARY DESTINATION "${INSTALL_LIB_DIR}"
		)
endif()
if(NOT SKIP_INSTALL_HEADERS AND NOT SKIP_INSTALL_ALL)
	install(FILES ${ZLIB_ANDROID_PUBLIC_HEADERS} DESTINATION "${INSTALL_INC_DIR}")
endif()
if(NOT SKIP_INSTALL_FILES AND NOT SKIP_INSTALL_ALL)
	install(FILES zlib.3 DESTINATION "${INSTALL_MAN_DIR}/man3")
endif()
if(NOT SKIP_INSTALL_FILES AND NOT SKIP_INSTALL_ALL)
	install(FILES ${ZLIB_ANDROID_PC} DESTINATION "${INSTALL_PKGCONFIG_DIR}")
endif()


