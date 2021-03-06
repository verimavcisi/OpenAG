cmake_minimum_required(VERSION 3.2)
project(OpenAG)

include("CMake/helpers.cmake")

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Set up some compiler-related things
if(MSVC)
	msvc_set_static_runtime()
	msvc_fix_xp_build()
	msvc_enable_lto()
else(MSVC)
	# Ensure 32-bit
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -m32")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -m32")

	# LTO doesn't always work out of the box, so hide it behind an option
	option(ENABLE_LTO "Enable Link-Time Optimizations" OFF)
	if(ENABLE_LTO)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -flto")
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -flto")
		set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -flto")
	endif(ENABLE_LTO)
endif(MSVC)

# Set up current_version.h
find_package(Git)
if(GIT_FOUND)
	message(STATUS "Found git at ${GIT_EXECUTABLE}")
	execute_process(COMMAND "${GIT_EXECUTABLE}" show -s --format=%ct
			OUTPUT_VARIABLE CURRENT_VERSION
			OUTPUT_STRIP_TRAILING_WHITESPACE)
	set(CURRENT_VERSION "${CURRENT_VERSION}ull")
else(GIT_FOUND)
	set(CURRENT_VERSION UINT64_MAX)
endif(GIT_FOUND)

configure_file(cl_dll/current_version.h.in current_version.h)

# Add discord-rpc
set(BUILD_EXAMPLES OFF)
set(CLANG_FORMAT_SUFFIX "-dummy")
add_subdirectory(external/discord-rpc)

# Collect the source files
add_subdirectory(cl_dll)
add_subdirectory(common)
add_subdirectory(dlls)
add_subdirectory(game_shared)
add_subdirectory(pm_shared)
add_subdirectory(public)

get_property(SOURCE_FILES GLOBAL PROPERTY SOURCE_FILES)
set(SOURCE_FILES ${SOURCE_FILES} "${CMAKE_BINARY_DIR}/current_version.h")

set(CMAKE_SHARED_LIBRARY_PREFIX  "")

add_library(client SHARED ${SOURCE_FILES})
target_include_directories(client PRIVATE
	${CMAKE_BINARY_DIR}
	common
	dlls
	engine
	external
	external/discord-rpc/include
	game_shared
	pm_shared
	public
	utils/vgui/include)

# Set up some additional compile flags and definitions
if(MSVC)
	target_compile_options(client PRIVATE
		# Disable some warnings which would take too much effort to fix
		/wd4018 # Signed/unsigned mismatch
		/wd4838 # Conversion from double to float
		/wd4996 # The POSIX name for this item is deprecated

		# Treat warnings as errors
		/WX)
else(MSVC)
	target_compile_options(client PRIVATE
		# High warning level and treat warnings as errors
		-Wall
		-Werror

		# Disable some warnings which would take too much effort to fix
		-Wno-parentheses
		-Wno-sign-compare
		-Wno-unused-function
		-Wno-unused-variable

		# Use the same FP instructions as others
		-march=pentium-m
		-mfpmath=387
		-mno-sse

		# Some additional flags from HLSDK
		-fno-strict-aliasing
		-fno-exceptions)

	# Some warnings are different between GCC and Clang
	if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
		target_compile_options(client PRIVATE
			-Wno-overloaded-virtual
			-Wno-unused-private-field
			-Wno-new-returns-null)
	else(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
		target_compile_options(client PRIVATE
			-Wno-unused-but-set-variable)
	endif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")

	target_compile_definitions(client PRIVATE
		-DGNUC
		-DPOSIX
		-D_POSIX
		-Dstricmp=strcasecmp
		-Dstrnicmp=strncasecmp
		-D_strnicmp=strncasecmp
		-D_snprintf=snprintf
		-D_snwprintf=swprintf
		-D_vsnprintf=vsnprintf
		-D_alloca=alloca)

	if(UNIX)
		target_compile_definitions(client PRIVATE
			-DLINUX
			-D_LINUX)
	endif(UNIX)
endif(MSVC)

if(WIN32)
	target_compile_definitions(client PRIVATE
		-D_CRT_SECURE_NO_WARNINGS
		-DWINDOWS
		-D_WINDOWS)

	target_link_libraries(client
		opengl32
		${CMAKE_SOURCE_DIR}/lib/public/SDL2.lib
		${CMAKE_SOURCE_DIR}/lib/public/game_controls.lib
		${CMAKE_SOURCE_DIR}/lib/public/libcurl_a.lib
		${CMAKE_SOURCE_DIR}/utils/vgui/lib/win32_vc6/vgui.lib)
elseif(UNIX)
	target_compile_definitions(client PRIVATE
		-DDISABLE_JUMP_ORIGIN
		-DDISABLE_VEC_ORIGIN)

	# I wasn't able to do this in any other way.
	# Things like imported targets result in linking with relative path.
	set_property(TARGET client PROPERTY LINK_FLAGS
		"-L${CMAKE_SOURCE_DIR}/linux/release -l:vgui.so")
endif(WIN32)

target_compile_definitions(client PRIVATE
	-DCLIENT_DLL
	-DCLIENT_WEAPONS
	-DUPDATE_CHECK)

target_link_libraries(client discord-rpc)
