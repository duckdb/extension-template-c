cmake_minimum_required(VERSION 2.8)

###
# Configuration
###
if(NOT DEFINED MINIMUM_DUCKDB_VERSION_MAJOR OR
        NOT DEFINED MINIMUM_DUCKDB_VERSION_MINOR OR
        NOT DEFINED MINIMUM_DUCKDB_VERSION_PATCH)
    message(FATAL_ERROR "DuckDB minimum version is required")
endif()
if(NOT DEFINED EXTENSION_NAME)
    message(FATAL_ERROR "DuckDB extension name is required")
endif()
add_definitions(-DDUCKDB_EXTENSION_API_VERSION_MAJOR=${MINIMUM_DUCKDB_VERSION_MAJOR})
add_definitions(-DDUCKDB_EXTENSION_API_VERSION_MINOR=${MINIMUM_DUCKDB_VERSION_MINOR})
add_definitions(-DDUCKDB_EXTENSION_API_VERSION_PATCH=${MINIMUM_DUCKDB_VERSION_PATCH})
add_definitions(-DDUCKDB_EXTENSION_NAME=${EXTENSION_NAME})

###
# Build
###
project(${EXTENSION_NAME} LANGUAGES C)

# Create Extension library
set(EXTENSION_SOURCES
        src/add_numbers.c
        src/capi_quack.c
)

if (EMSCRIPTEN)
	add_library(${EXTENSION_NAME} STATIC ${EXTENSION_SOURCES})
else()
	add_library(${EXTENSION_NAME} SHARED ${EXTENSION_SOURCES})
endif()

# Hide symbols
set(CMAKE_CXX_VISIBILITY_PRESET hidden)
set(CMAKE_C_VISIBILITY_PRESET hidden)
set(VISIBILITY_INLINES_HIDDEN ON)
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -s")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -s")

# Include own headers
target_include_directories(${EXTENSION_NAME} PRIVATE src/include)

# Include DuckDB C API headers
target_include_directories(${EXTENSION_NAME} PRIVATE duckdb_capi)