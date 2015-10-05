cmake_minimum_required(VERSION 3.0)

project(protobuf VERSION 3.0.0)

option(BUILD_TESTING "Build tests" OFF)
option(BUILD_SHARED_LIBS "Build Shared Libraries" OFF)
option(USE_ZLIB "Build with zlib support" OFF)
if(MSVC)
	OPTION(MSVC_LINK_STATIC_RUNTIME "Link with msvc static runtime" OFF)
endif()

# debug libraries can coexist with release libraries, so make debug names different
set(CMAKE_DEBUG_POSTFIX "d")

# Path to main configure script
set(protobuf_CONFIGURE_SCRIPT "../configure.ac")

# Parse version from configure script
file(STRINGS "${protobuf_CONFIGURE_SCRIPT}" protobuf_VERSION_LINE
  LIMIT_COUNT 1
  REGEX "^AC_INIT")
# Replace special characters
string(REPLACE "(" "_" protobuf_VERSION_LINE ${protobuf_VERSION_LINE})
string(REPLACE ")" "_" protobuf_VERSION_LINE ${protobuf_VERSION_LINE})
string(REPLACE "[" "_" protobuf_VERSION_LINE ${protobuf_VERSION_LINE})
string(REPLACE "]" "_" protobuf_VERSION_LINE ${protobuf_VERSION_LINE})
# Parse version string
string(REGEX REPLACE "^AC_INIT__Protocol Buffers_,_([^_]+).*$" "\\1"
    protobuf_VERSION_STRING "${protobuf_VERSION_LINE}")
# Parse version tweaks
string(REGEX REPLACE "^([0-9]+)\\.([0-9]+)\\.([0-9]+).*$" "\\1"
  protobuf_VERSION_MAJOR "${protobuf_VERSION_STRING}")
string(REGEX REPLACE "^([0-9]+)\\.([0-9]+)\\.([0-9]+).*$" "\\2"
  protobuf_VERSION_MINOR "${protobuf_VERSION_STRING}")
string(REGEX REPLACE "^([0-9]+)\\.([0-9]+)\\.([0-9]+).*$" "\\3"
  protobuf_VERSION_PATCH "${protobuf_VERSION_STRING}")
# Package version
set(protobuf_VERSION
  "${protobuf_VERSION_MAJOR}.${protobuf_VERSION_MINOR}.${protobuf_VERSION_PATCH}")

add_definitions(-DGOOGLE_PROTOBUF_CMAKE_BUILD)

find_package(Threads REQUIRED)
if(CMAKE_USE_PTHREADS_INIT)
	add_definitions(-DHAVE_PTHREAD)
endif()

# statically link msvc runtime
if(MSVC AND MSVC_LINK_STATIC_RUNTIME)
	message(STATUS "Linking runtime statically")
	foreach(flag_var
		CMAKE_C_FLAGS CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE
		CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELWITHDEBINFO
		CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
		CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)

		if(${flag_var} MATCHES "/MD")
			string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
		endif(${flag_var} MATCHES "/MD")
	endforeach(flag_var)
endif()

if(USE_ZLIB)
	# config flag tells cmake not to use the build-in package file to find zlib but
	# switch to config mode where it tries to find the config file provided by
	# the package
	find_package(zlib REQUIRED CONFIG)
	list(APPEND compile_definitions "HAVE_ZLIB")
	list(APPEND additional_libs "zlib")
endif()

if (MSVC)
	list(APPEND compile_flags
		"/wd4244 /wd4267 /wd4018 /wd4355 /wd4800 /wd4251 /wd4996 /wd4146 /wd4305")

	string(REPLACE "/" "\\" PROTOBUF_SOURCE_WIN32_PATH ${protobuf_SOURCE_DIR})
	string(REPLACE "/" "\\" PROTOBUF_BINARY_WIN32_PATH ${protobuf_BINARY_DIR})
	configure_file(extract_includes.bat.in extract_includes.bat)
endif()

# strips the cmake part in the path
get_filename_component(protobuf_source_dir ${protobuf_SOURCE_DIR} PATH)
set(protobuf_public_hdr_dir "${protobuf_source_dir}/src")

add_subdirectory(libprotobuf-lite)
add_subdirectory(libprotobuf)
add_subdirectory(libprotoc)
add_subdirectory(protoc)

if (BUILD_TESTING)
	include(tests.cmake)
endif()

install(DIRECTORY "${protobuf_source_dir}/src/google"
	DESTINATION "include" COMPONENT dev
	FILES_MATCHING
	PATTERN "*.h")

install(EXPORT ${CMAKE_PROJECT_NAME}Config
	DESTINATION lib/cmake)
