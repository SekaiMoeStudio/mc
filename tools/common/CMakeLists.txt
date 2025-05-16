# Common helper tools CMake configuration

# Source files
set(COMMON_LIB_SOURCES
    toml.cpp
    path.cpp
    debug.cpp
    jobserver.cpp
)

# Create static library
add_library(common_lib STATIC ${COMMON_LIB_SOURCES})

# Set compile options
target_compile_options(common_lib PRIVATE
    -Wall
    -std=c++14
    -g
    -O2
    ${CXXFLAGS_EXTRA}
)

# Set library output directory
set_target_properties(common_lib PROPERTIES
    ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
)

# Generate compile_commands.json for IDE support
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
