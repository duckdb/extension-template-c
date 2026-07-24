cmake_minimum_required(VERSION 3.5...3.29)
option(DUCKDB_WASM_EXTENSION "Whether compiling for Wasm target" OFF)

###
# Configuration
###
if(NOT DEFINED EXTENSION_NAME)
    message(FATAL_ERROR "DuckDB extension name is required")
endif()
if (DEFINED TARGET_DUCKDB_VERSION_MAJOR)
    add_definitions(-DDUCKDB_EXTENSION_API_VERSION_MAJOR=${TARGET_DUCKDB_VERSION_MAJOR})
endif()
if (DEFINED TARGET_DUCKDB_VERSION_MINOR)
    add_definitions(-DDUCKDB_EXTENSION_API_VERSION_MINOR=${TARGET_DUCKDB_VERSION_MINOR})
endif()
if (DEFINED TARGET_DUCKDB_VERSION_PATCH)
    add_definitions(-DDUCKDB_EXTENSION_API_VERSION_PATCH=${TARGET_DUCKDB_VERSION_PATCH})
endif()

add_definitions(-DDUCKDB_EXTENSION_NAME=${EXTENSION_NAME})

if (DEFINED DUCKDB_EXTENSION_API_VERSION_UNSTABLE)
    add_definitions(-DDUCKDB_EXTENSION_API_VERSION_UNSTABLE=${DUCKDB_EXTENSION_API_VERSION_UNSTABLE})
endif()

###
# Build
###
project(${EXTENSION_NAME} LANGUAGES C)

# Create Extension library
set(EXTENSION_SOURCES
        src/add_numbers.c
        src/capi_quack.c
)

# When compiled in-tree as part of a DuckDB build (via duckdb_extension_load(... SOURCE_DIR ...)),
# DuckDB provides the C-API build helpers. Build through them so the static target is compiled with
# DUCKDB_BUILD_STATIC_EXTENSION and tagged DUCKDB_EXTENSION_KIND=CAPI, letting DuckDB's generated
# loader statically link and auto-register the extension - no separate wrapper CMakeLists needed.
# Stable vs unstable is driven by the same variables the standalone build uses:
# DUCKDB_EXTENSION_API_VERSION_UNSTABLE selects the unstable API, otherwise the stable API is targeted
# at TARGET_DUCKDB_VERSION_*. In-tree neither is set by default, so it falls back to the unstable
# helpers, which target the exact DuckDB being built.
if(COMMAND build_static_extension_capi_unstable)
    if(DEFINED DUCKDB_EXTENSION_API_VERSION_UNSTABLE OR NOT DEFINED TARGET_DUCKDB_VERSION_MAJOR)
        build_loadable_extension_capi_unstable(${EXTENSION_NAME} ${EXTENSION_SOURCES})
        build_static_extension_capi_unstable(${EXTENSION_NAME} ${EXTENSION_SOURCES})
    else()
        # TARGET_DUCKDB_VERSION_MINOR/PATCH may be unset - either from a partial standalone
        # TARGET_DUCKDB_VERSION (e.g. "v1" or "v1.2"), or from a duckdb_extension_load() caller that
        # only defines a partial version. The stable helpers take MAJOR MINOR PATCH positionally, so
        # default the missing parts to 0 (semver: "v1" -> 1.0.0).
        if(NOT DEFINED TARGET_DUCKDB_VERSION_MINOR)
            set(TARGET_DUCKDB_VERSION_MINOR 0)
        endif()
        if(NOT DEFINED TARGET_DUCKDB_VERSION_PATCH)
            set(TARGET_DUCKDB_VERSION_PATCH 0)
        endif()
        build_loadable_extension_capi(${EXTENSION_NAME} ${TARGET_DUCKDB_VERSION_MAJOR} ${TARGET_DUCKDB_VERSION_MINOR} ${TARGET_DUCKDB_VERSION_PATCH} ${EXTENSION_SOURCES})
        build_static_extension_capi(${EXTENSION_NAME} ${TARGET_DUCKDB_VERSION_MAJOR} ${TARGET_DUCKDB_VERSION_MINOR} ${TARGET_DUCKDB_VERSION_PATCH} ${EXTENSION_SOURCES})
    endif()
    foreach(_capi_target ${EXTENSION_NAME}_loadable_extension ${EXTENSION_NAME}_extension)
        target_include_directories(${_capi_target} PRIVATE src/include)
    endforeach()
    return()
endif()

if (DUCKDB_WASM_EXTENSION)
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
