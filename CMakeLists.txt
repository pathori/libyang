cmake_minimum_required(VERSION 2.6)
project(libyang C)

set(LIBYANG_DESCRIPTION "libyang is YANG data modelling language parser and toolkit written (and providing API) in C.")

# set version
set(LIBYANG_MAJOR_VERSION 0)
set(LIBYANG_MINOR_VERSION 9)
set(LIBYANG_MICRO_VERSION 149)
set(LIBYANG_VERSION ${LIBYANG_MAJOR_VERSION}.${LIBYANG_MINOR_VERSION}.${LIBYANG_MICRO_VERSION})
set(LIBYANG_SOVERSION ${LIBYANG_MAJOR_VERSION}.${LIBYANG_MINOR_VERSION})

# include custom Modules
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/CMakeModules/")

find_package(FLEX)
find_package(BISON)
find_program (SED_TOOL NAMES sed)

if(NOT BISON_FOUND)
	message(WARNING "Missing Bison.\nTarget bison does not exist.")
elseif (NOT FLEX_FOUND)
	message(WARNING "Missing Flex.\nTarget bison does not exist.")
else()
	if (BISON_VERSION VERSION_LESS 3)
		set(EMPTYDIR "")
	else ()
		set(EMPTYDIR "%empty")
	endif()
	configure_file(${PROJECT_SOURCE_DIR}/src/yang.y.in ${PROJECT_SOURCE_DIR}/src/yang.y)
	add_custom_target(bison
		WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/src
		COMMAND bison -Wno-other -l -o parser_yang_bis.c --defines=parser_yang_bis.h yang.y
		COMMAND flex -L -o parser_yang_lex.c --header-file=parser_yang_lex.h yang.l
		COMMAND ${SED_TOOL} -i 's/int yychar\;/int yychar\;\\nchar *s = NULL, *tmp_s = NULL\;\\nchar rev[LY_REV_SIZE]\;\\nstruct lys_module *trg = NULL\;\\nstruct lys_node *tpdf_parent = NULL, *data_node = NULL\;\\nvoid *actual = NULL\;\\nint config_inherit = 0, actual_type = 0\;\\nint64_t cnt_val\;/' parser_yang_bis.c
	)
endif()

# check the supported platform
if(NOT UNIX)
	message(FATAL_ERROR "Only *nix like systems are supported.")
endif()

if(NOT LIB_INSTALL_DIR)
	set(LIB_INSTALL_DIR lib)
endif()

if(NOT INCLUDE_INSTALL_DIR)
	set(INCLUDE_INSTALL_DIR include/libyang)
endif()

# set default build type if not specified by user
if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE debug)
endif()

set(CMAKE_C_FLAGS         "${CMAKE_C_FLAGS} -Wall -Wextra -fvisibility=hidden")
set(CMAKE_C_FLAGS_RELEASE "-O2 -DNDEBUG")
set(CMAKE_C_FLAGS_DEBUG   "-g -O0")

if(CMAKE_BUILD_TYPE STREQUAL debug)
	option(ENABLE_BUILD_TESTS "Build tests" ON)
	option(ENABLE_VALGRIND_TESTS "Build tests with valgrind" ON)
else()
	option(ENABLE_BUILD_TESTS "Build tests" OFF)
	option(ENABLE_VALGRIND_TESTS "Build tests with valgrind" OFF)
endif()

set(libsrc
	src/common.c
	src/context.c
	src/log.c
	src/dict.c
	src/resolve.c
	src/validation.c
	src/xml.c
	src/parser.c
	src/parser_yin.c
	src/parser_xml.c
	src/parser_json.c
	src/parser_yang_bis.c
	src/parser_yang_lex.c
	src/parser_yang.c
	src/tree_schema.c
	src/tree_data.c
	src/printer.c
	src/xpath.c
	src/printer_yang.c
	src/printer_yin.c
	src/printer_xml.c
	src/printer_tree.c
	src/printer_info.c
	src/printer_json.c
	src/yang_types.c)

set(lintsrc
	tools/lint/main.c
	tools/lint/commands.c
	tools/lint/completion.c
	linenoise/linenoise.c)

set(headers
	src/libyang.h
	src/tree_schema.h
	src/tree_data.h
	src/xml.h
	src/dict.h)

add_library(yang SHARED ${libsrc})
set_target_properties(yang PROPERTIES VERSION ${LIBYANG_VERSION} SOVERSION ${LIBYANG_SOVERSION})

# link math
target_link_libraries(yang m)

# find pthreads
set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
find_package(Threads REQUIRED)
target_link_libraries(yang ${CMAKE_THREAD_LIBS_INIT})

# find PCRE library
find_package(PCRE REQUIRED)
include_directories(${PCRE_INCLUDE_DIRS})
target_link_libraries(yang ${PCRE_LIBRARIES})

install(TARGETS yang DESTINATION ${LIB_INSTALL_DIR})
install(FILES ${headers} DESTINATION ${INCLUDE_INSTALL_DIR})

find_package(PkgConfig)
if(PKG_CONFIG_FOUND)
	# generate and install pkg-config file
	configure_file("libyang.pc.in" "libyang.pc" @ONLY)
	install(FILES "${CMAKE_CURRENT_BINARY_DIR}/libyang.pc" DESTINATION "${CMAKE_INSTALL_PREFIX}/${LIB_INSTALL_DIR}/pkgconfig")
	# check that pkg-config includes the used path
	execute_process(COMMAND ${PKG_CONFIG_EXECUTABLE} --variable pc_path pkg-config RESULT_VARIABLE RETURN OUTPUT_VARIABLE PC_PATH ERROR_QUIET)
	if(RETURN EQUAL 0)
		string(REGEX MATCH "${CMAKE_INSTALL_PREFIX}/${LIB_INSTALL_DIR}/pkgconfig" SUBSTR "${PC_PATH}")
		string(LENGTH "${SUBSTR}" SUBSTR_LEN)
		if(SUBSTR_LEN EQUAL 0)
			message(WARNING "pkg-config will not detect the new package after installation, adjust PKG_CONFIG_PATH using \"export PKG_CONFIG_PATH=\${PKG_CONFIG_PATH}:${CMAKE_INSTALL_PREFIX}/${LIB_INSTALL_DIR}/pkgconfig\".")
		endif()
	endif()
endif()

# generate doxygen documentation for libyang API
find_package(Doxygen)
if(DOXYGEN_FOUND)
	set(DOXYGEN_SKIP_DOT TRUE)
	add_custom_target(doc
			COMMAND ${DOXYGEN_EXECUTABLE} ${CMAKE_BINARY_DIR}/Doxyfile
			WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
	configure_file(Doxyfile.in Doxyfile)
endif()

# clean cmake cache
add_custom_target(cclean
		COMMAND make clean
		COMMAND find . -iname '*cmake*' -not -name CMakeLists.txt -not -path './CMakeModules*' -exec rm -rf {} +
		COMMAND rm -rf Makefile Doxyfile
		WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})

add_executable(yanglint ${lintsrc})
target_link_libraries(yanglint yang)

if(ENABLE_VALGRIND_TESTS)
	set(ENABLE_BUILD_TESTS ON)
endif()

if(ENABLE_BUILD_TESTS)
	find_package(CMocka 1.0.0)
	if(CMOCKA_FOUND)
		enable_testing()
		add_subdirectory(tests)
	endif(CMOCKA_FOUND)
endif(ENABLE_BUILD_TESTS)

if(JAVASCRIPT_BINDING)
	include(swig/javascript/CMakeLists.txt)
endif()
