cmake_minimum_required(VERSION 3.5)

include(GenerateExportHeader)

# To make frugen 32-bit, set BINARY_32BIT or use an external toolchain file
option(BINARY_32BIT "compile 32bit version" OFF)
if(BINARY_32BIT)
	# Note: this have to go before PROJECT
	set(CMAKE_C_COMPILER gcc -m32)
endif(BINARY_32BIT)

execute_process(COMMAND
	git
	describe
	--always
	--long
	--dirty
	WORKING_DIRECTORY
	"${CMAKE_CURRENT_SOURCE_DIR}"
	RESULT_VARIABLE
	res
	OUTPUT_VARIABLE
	gitver
	ERROR_QUIET
	OUTPUT_STRIP_TRAILING_WHITESPACE
)
if(NOT res EQUAL 0)
	set(gitver "v3.0-dirty-orphan")
endif()

string(REPLACE "v" "" gitver "${gitver}")
string(REPLACE "-" "." gitver "${gitver}")

# This is the version of the project w/o hash information
string(REGEX REPLACE "((\\.?[0-9]+)+).*" "\\1" gitver_short "${gitver}")
project(frugen VERSION ${gitver_short} LANGUAGES C)

option(BUILD_SHARED_LIB "build shared library" ON)
option(BINARY_STATIC "link all libs statically when compiling frugen" OFF)
option(LIBFRU_STATIC "link libfru statically when compiling frugen" OFF)
option(ENABLE_JSON "enable JSON support" ON)
option(JSON_STATIC "link json-c library statically when compiling frugen" OFF)
option(DEBUG_OUTPUT "show extra debug output" OFF)

message(STATUS "Compiler is ${CMAKE_C_COMPILER_ID} ${CMAKE_C_COMPILER_VERSION}")
if(MSVC)
	# warning level 4
	add_compile_options(/W4)
else(MSVC)
	set(CMAKE_C_FLAGS_RELEASE "-Os")
	set(CMAKE_C_FLAGS_DEBUG "-g3 -O0")
	# Enable static analyzer for gcc 13+
	# It still may produce some falso positives, but it's quite decent already.
	if(CMAKE_C_COMPILER_ID STREQUAL "GNU" AND CMAKE_C_COMPILER_VERSION VERSION_GREATER_EQUAL "13.0")
		add_compile_options(-fanalyzer)
	endif()
	# additional warnings
	add_compile_options(
		-Wall
		-Werror
		-Wfatal-errors
		-Wextra
		-Wunreachable-code
		-Wunused-macros
		-Wunused-local-typedefs
		-ffunction-sections
		-fdata-sections
	)
endif(MSVC)

if(DEBUG_OUTPUT)
	add_definitions(-DDEBUG)
endif(DEBUG_OUTPUT)

add_definitions(-DVERSION="${gitver}")

configure_file(fru.h.in fru.h @ONLY)

include("doxygen.cmake")
set(DOXY_PROJECT_NUMBER "${gitver}")
set(DOXY_OUTPUT_LANGUAGE "English")
set(DOXY_INPUT "${PROJECT_SOURCE_DIR} ${CMAKE_BINARY_DIR}")
set(DOXY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/docs")
set(DOXY_STRIP_FROM_PATH "${CMAKE_BINARY_DIR}")
add_documentation(docs Doxyfile)

# build libfru
set(libfru_SOURCES
	lib/fru__decode.c
	lib/fru_enable_area.c
	lib/fru_errno.c
	lib/fru_add_custom.c
	lib/fru_add_mr.c
	lib/fru_common.c
	lib/fru_delete_custom.c
	lib/fru_get_custom.c
	lib/fru_init.c
	lib/fru_internal.c
	lib/fru_load.c
	lib/fru_mr_ops.c
	lib/fru_save.c
	lib/fru_setfield.c
	lib/fru_setfield_binary.c
	lib/fru_getfield.c
)
set(libfru_HEADERS
	${CMAKE_CURRENT_BINARY_DIR}/fru.h
	fru_errno.h
	lib/fru-private.h
)

set(libfru_PUBLIC_HEADERS
	${CMAKE_CURRENT_BINARY_DIR}/fru.h
	fru_errno.h
)

if(BUILD_SHARED_LIB)
	add_library(fru-shared SHARED ${libfru_SOURCES} ${libfru_HEADERS})
	set_target_properties(fru-shared PROPERTIES C_VISIBILITY_PRESET hidden)
	generate_export_header(fru-shared BASE_NAME fru)

	add_library(frugen::fru-shared ALIAS fru-shared)
	target_include_directories(fru-shared PUBLIC
		$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
		$<INSTALL_INTERFACE:include>
	)
	set_target_properties(fru-shared PROPERTIES OUTPUT_NAME fru CLEAN_DIRECT_OUTPUT 1)
	set_target_properties(fru-shared PROPERTIES VERSION ${PROJECT_VERSION} SOVERSION ${PROJECT_VERSION_MAJOR})
	set_target_properties(fru-shared PROPERTIES PUBLIC_HEADER "${libfru_PUBLIC_HEADERS}")
	set_target_properties(fru-shared PROPERTIES C_STANDARD 23)
	list(APPEND LIB_TARGETS "fru-shared")
endif(BUILD_SHARED_LIB)
if(BINARY_STATIC OR LIBFRU_STATIC OR NOT BUILD_SHARED_LIB)
	add_library(fru-static STATIC ${libfru_SOURCES} ${libfru_HEADERS})
	add_library(frugen::fru-static ALIAS fru-static)
	set_target_properties(fru-static PROPERTIES C_VISIBILITY_PRESET hidden)
	generate_export_header(fru-static BASE_NAME fru)
	target_include_directories(fru-static PUBLIC
		$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
		$<INSTALL_INTERFACE:include>
	)
	set_target_properties(fru-static PROPERTIES OUTPUT_NAME fru CLEAN_DIRECT_OUTPUT 1)
	set_target_properties(fru-static PROPERTIES PUBLIC_HEADER "${CMAKE_CURRENT_BINARY_DIR}/fru.h")
	set_target_properties(fru-static PROPERTIES C_STANDARD 23)
	list(APPEND LIB_TARGETS "fru-static")
endif(BINARY_STATIC OR LIBFRU_STATIC OR NOT BUILD_SHARED_LIB)

# target for frugen
set(frugen_SOURCES frugen.c)
add_executable(frugen ${frugen_SOURCES})

set_target_properties(frugen PROPERTIES C_VISIBILITY_PRESET default)

if(ENABLE_JSON)
	find_package(PkgConfig)
	pkg_check_modules(json-c json-c)

# For debugging
#	get_cmake_property(_variableNames VARIABLES)
#	list (SORT _variableNames)
#	foreach (_variableName ${_variableNames})
#	    message(STATUS "VARIABLE ${_variableName}=${${_variableName}}")
#	endforeach()

	if(json-c_FOUND)
		if(JSON_STATIC OR BINARY_STATIC)
# TODO: Find a way to detect that static version is indeed present
#       as it is missing in Fedora by default
#			if (NOT json-c_STATIC_FOUND)
#				message (WARNING "No STATIC json-c library available")
#				set(ENABLE_JSON false)
#			else ()
				message(STATUS "Linking json-c library statically")
				target_link_libraries(frugen ${json-c_STATIC_LIBRARIES})
#			endif ()
		else ()
			message(STATUS "Using shared json-c library")
			target_link_libraries(frugen ${json-c_LIBRARIES})
		endif ()
	endif ()

	if (ENABLE_JSON)
		add_definitions(-D__HAS_JSON__)
		list(APPEND frugen_SOURCES frugen-json.c)
		set_target_properties(frugen PROPERTIES SOURCES "${frugen_SOURCES}")
		target_include_directories(frugen PRIVATE ${json-c_INCLUDE_DIRS})
	else ()
		message (WARNING "JSON library support *disabled*!")
	endif ()
endif(ENABLE_JSON)

list(APPEND ALL_TARGETS ${LIB_TARGETS} "frugen")

if(CMAKE_C_COMPILER_ID STREQUAL "GNU")
	if (CMAKE_BUILD_TYPE STREQUAL "Debug")
		set_target_properties(frugen PROPERTIES LINK_FLAGS "-Wl,--gc-sections,--print-gc-sections")
	else()
		set_target_properties(frugen PROPERTIES LINK_FLAGS "-Wl,--gc-sections")
	endif()
endif()

set_target_properties(frugen PROPERTIES C_STANDARD 23)

if(BINARY_STATIC)
	target_link_libraries(frugen -static)
	message(STATUS "Building a completely static frugen binary")
endif(BINARY_STATIC)
if(BINARY_STATIC OR LIBFRU_STATIC OR NOT BUILD_SHARED_LIB)
	target_link_libraries(frugen fru-static)
	message(STATUS "Linking fru library statically")
else()
	target_link_libraries(frugen fru-shared)
	message(STATUS "Using shared libfru library")
endif(BINARY_STATIC OR LIBFRU_STATIC OR NOT BUILD_SHARED_LIB)

# install targets
install(TARGETS frugen RUNTIME DESTINATION bin)
install(TARGETS ${LIB_TARGETS}
	EXPORT ${PROJECT_NAME}-targets
	PUBLIC_HEADER DESTINATION include
	ARCHIVE DESTINATION lib
	LIBRARY DESTINATION lib
	RUNTIME DESTINATION bin
)
install(EXPORT ${PROJECT_NAME}-targets
	NAMESPACE ${PROJECT_NAME}::
	FILE ${PROJECT_NAME}-config.cmake
	DESTINATION lib/cmake/${PROJECT_NAME}
)
