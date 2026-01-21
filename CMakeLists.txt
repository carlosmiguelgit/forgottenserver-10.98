cmake_minimum_required(VERSION 3.19)

set(CMAKE_DISABLE_SOURCE_CHANGES ON)
set(CMAKE_DISABLE_IN_SOURCE_BUILD ON)

if(DEFINED ENV{VCPKG_DEFAULT_TRIPLET} AND NOT DEFINED VCPKG_TARGET_TRIPLET)
	set(VCPKG_TARGET_TRIPLET "$ENV{VCPKG_DEFAULT_TRIPLET}" CACHE STRING "The vcpkg triplet")
endif()

project(tfs CXX)

set(VCPKG_MANIFEST_NO_DEFAULT_FEATURES ON)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

if(NOT WIN32)
	add_compile_options(-Wall -Wextra -Wnon-virtual-dtor -Wold-style-cast -pedantic -Werror -pipe -fvisibility=hidden)
endif()

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
	add_compile_options(-fno-strict-aliasing)
endif()

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
	add_compile_options(-Wimplicit-fallthrough -Wmove)
endif()


### INIT Find packages ###

# LIB Open SSL
find_package(OpenSSL REQUIRED)

# LIB FMT
find_package(fmt 8.1.1 CONFIG)
if(NOT fmt_FOUND)
	find_package(fmt 8.1.1 REQUIRED)
endif()

# LIB MySQL (Linux)
option(USE_LIBMYSQL "Use libmysql instead of libmariadb" ON)

if(WIN32)
	find_package(unofficial-libmariadb CONFIG REQUIRED)
	set(MYSQL_CLIENT_LIBS unofficial::libmariadb)
else()
	if(USE_LIBMYSQL)
		find_library(MYSQL_CLIENT_LIBS mysqlclient)
		if (NOT MYSQL_CLIENT_LIBS)
			message(FATAL_ERROR "libmysqlclient não encontrada. Instale com: sudo apt install libmysqlclient-dev")
		endif()
	else()
		find_package(unofficial-libmariadb CONFIG REQUIRED)
		set(MYSQL_CLIENT_LIBS unofficial::libmariadb)
	endif()
endif()


# LIB Threads
find_package(Threads REQUIRED)

# LIB PugiXML
find_package(PugiXML CONFIG REQUIRED)

# LIB LuaJIT or Lua
option(USE_LUAJIT "Use LuaJIT" OFF)

if(USE_LUAJIT)
	list(APPEND VCPKG_MANIFEST_FEATURES "luajit")
	find_package(LuaJIT REQUIRED)
else()
	list(APPEND VCPKG_MANIFEST_FEATURES "lua")
	find_package(Lua REQUIRED)
endif()

# LIB for MAC
if(APPLE)
	find_package(Iconv REQUIRED)
endif()

# LIB Locale & Boost
set(BOOST_REQUIRED_COMPONENTS system iostreams locale)
set(BOOST_REQUIRED_VERSION 1.71.0)
find_package(Boost ${BOOST_REQUIRED_VERSION} REQUIRED COMPONENTS ${BOOST_REQUIRED_COMPONENTS})

### END find packages ###


include_directories(
	${Boost_INCLUDE_DIRS}
	${OPENSSL_INCLUDE_DIR}
	${LUA_INCLUDE_DIR}
	${PUGIXML_INCLUDE_DIR}
)

if(USE_LIBMYSQL)
	include_directories(/usr/include/mysql)
endif()

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Forçar MT (static runtime) para Windows
if(MSVC)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

add_subdirectory(src)

add_executable(tfs ${tfs_MAIN})

target_link_libraries(tfs
    tfslib
    ${MYSQL_CLIENT_LIBS}
    ${Boost_LIBRARIES}
    Threads::Threads
    OpenSSL::Crypto
    fmt::fmt
    ${PUGIXML_LIB}
    ${LUA_LIBRARIES}
)

### INIT INTERPROCEDURAL_OPTIMIZATION ###
include(CheckIPOSupported)
check_ipo_supported(RESULT result OUTPUT error)

if(result)
	set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
else()
	message(STATUS "IPO is not supported: ${error}")
endif()
### END INTERPROCEDURAL_OPTIMIZATION ###


### INIT Git Version ###
option(SKIP_GIT "Skip checking for git updates" ON)

if(NOT SKIP_GIT)
	set(PRE_CONFIGURE_FILE "cmake/gitmetadata.h.in")
	set(POST_CONFIGURE_FILE "${CMAKE_CURRENT_BINARY_DIR}/gitmetadata.h")
	include(git_watcher)
	if(Git_FOUND)
		add_dependencies(tfs check_git)
		target_include_directories(tfs PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
	endif()
endif()
### END Git Version ###

target_precompile_headers(tfs PUBLIC src/otpch.h)
