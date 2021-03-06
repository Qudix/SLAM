cmake_minimum_required(VERSION 3.18)

# ---- Project ----

project(
	SLAM
	VERSION 0.0.0
	LANGUAGES CXX
)

configure_file(
	${CMAKE_CURRENT_SOURCE_DIR}/cmake/Version.h.in
	${CMAKE_CURRENT_BINARY_DIR}/include/Version.h
	@ONLY
)

configure_file(
	${CMAKE_CURRENT_SOURCE_DIR}/cmake/version.rc.in
	${CMAKE_CURRENT_BINARY_DIR}/version.rc
	@ONLY
)

# ---- Include guards ----

if(PROJECT_SOURCE_DIR STREQUAL PROJECT_BINARY_DIR)
	message(
		FATAL_ERROR
			"In-source builds not allowed. Please make a new directory (called a build directory) and run CMake from there."
)
endif()

# ---- Globals ----

set_property(GLOBAL PROPERTY PREDEFINED_TARGETS_FOLDER "CMake")
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

#set(CMAKE_CONFIGURATION_TYPES "Debug;RelWithDebInfo")

# ---- Dependencies ----

set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_DEBUG OFF)

if (MSVC)
	add_compile_definitions(
		_UNICODE
	)

	set(COMPILE_OPTIONS_DEBUG
		/Zc:inline-	# Keep unreferenced COMDAT
		/JMC	# Just My Code debugging
	)

	set(COMPILE_OPTIONS_RELEASE
		/Zc:inline	# Remove unreferenced COMDAT
		/JMC-	# Disable Just My Code debugging
	)

	add_compile_options(
		/Gy	# Enable Function-Level Linking
		/MP	# Build with Multiple Processes
		/Oi	# Generate Intrinsic Functions
		/sdl	# Enable Additional Security Checks
		/Zi	# Debug Information Format

		/permissive-	# Standards conformance

		/Zc:alignedNew	# C++17 over-aligned allocation
		/Zc:auto	# Deduce Variable Type
		/Zc:char8_t
		/Zc:__cplusplus	# Enable updated __cplusplus macro
		/Zc:externC
		/Zc:externConstexpr	# Enable extern constexpr variables
		/Zc:forScope	# Force Conformance in for Loop Scope
		/Zc:hiddenFriend
		/Zc:implicitNoexcept	# Implicit Exception Specifiers
		/Zc:lambda
		/Zc:noexceptTypes	# C++17 noexcept rules
		/Zc:preprocessor	# Enable preprocessor conformance mode
		/Zc:referenceBinding	# Enforce reference binding rules
		/Zc:rvalueCast	# Enforce type conversion rules
		/Zc:sizedDealloc	# Enable Global Sized Deallocation Functions
		/Zc:strictStrings	# Disable string literal type conversion
		/Zc:ternary	# Enforce conditional operator rules
		/Zc:threadSafeInit	# Thread-safe Local Static Initialization
		/Zc:tlsGuards
		/Zc:trigraphs	# Trigraphs Substitution
		/Zc:wchar_t	# wchar_t Is Native Type

		/experimental:external
		/external:anglebrackets
		/external:W0

		/W4	# Warning level (all warnings)
		/WX	# Warning level (warnings are errors)

		"$<$<CONFIG:Debug>:${COMPILE_OPTIONS_DEBUG}>"
		"$<$<CONFIG:Release>:${COMPILE_OPTIONS_RELEASE}>"
	)

	set(LINK_OPTIONS_DEBUG
		/INCREMENTAL	# Link Incrementally
		/OPT:NOREF	# Optimizations (keep functions/data never referenced)
		/OPT:NOICF	# Optimizations (prevent identical COMDAT folding)
	)

	set(LINK_OPTIONS_RELEASE
		/INCREMENTAL:NO	# Link Incrementally
		/OPT:REF	# Optimizations (eliminate functions/data never referenced)
		/OPT:ICF	# Optimizations (perform identical COMDAT folding)
	)

	add_link_options(
		"$<$<CONFIG:Debug>:${LINK_OPTIONS_DEBUG}>"
		"$<$<CONFIG:Release>:${LINK_OPTIONS_RELEASE}>"
	)
endif()

add_subdirectory(extern/CommonLibSSE CommonLibSSE)

find_package(spdlog REQUIRED)

# ---- Add source files ----

include(cmake/headerlist.cmake)
include(cmake/sourcelist.cmake)

source_group(
	TREE ${CMAKE_CURRENT_SOURCE_DIR}
	FILES
		${headers}
		${sources}
)

source_group(
	TREE ${CMAKE_CURRENT_BINARY_DIR}
	FILES
		${CMAKE_CURRENT_BINARY_DIR}/include/Version.h
)

# ---- Create DLL ----

add_library(${PROJECT_NAME} SHARED
	${headers}
	${sources}
	${CMAKE_CURRENT_BINARY_DIR}/include/Version.h
	${CMAKE_CURRENT_BINARY_DIR}/version.rc
	.clang-format
)

set_property(
	TARGET
		${PROJECT_NAME}
		CommonLibSSE
	PROPERTY
		MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>"
)

target_compile_features(${PROJECT_NAME}
	PUBLIC
		cxx_std_17
)

target_compile_options(${PROJECT_NAME}
	PRIVATE
		"$<$<BOOL:${MSVC}>:/TP>"
)

target_include_directories(${PROJECT_NAME}
	PRIVATE
		$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
        $<INSTALL_INTERFACE:src>
)

target_link_libraries(${PROJECT_NAME}
	PUBLIC
		CommonLibSSE::CommonLibSSE
		spdlog::spdlog
)

target_link_options(${PROJECT_NAME}
	PUBLIC
		${LINK_OPTIONS_${CONFIG}}
)

target_precompile_headers(${PROJECT_NAME}
	PRIVATE
		src/PCH.h
)

# ---- Post build ----

add_custom_command(
	TARGET ${PROJECT_NAME}
	POST_BUILD
	COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_FILE:${PROJECT_NAME}> $ENV{SkyrimSEPath}/Data/SKSE/Plugins/
	#COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_PDB_FILE:${PROJECT_NAME}> $ENV{SkyrimSEPath}/Data/SKSE/Plugins/
)