# Mini version of cargo CMake configuration
cmake_minimum_required(VERSION 3.14)

# Source files
set(MINICARGO_SOURCES
    main.cpp
    manifest.cpp
    repository.cpp
    cfg.cpp
    build.cpp
    jobs.cpp
    file_timestamp.cpp
    os.cpp
)

# Create executable
add_executable(minicargo ${MINICARGO_SOURCES})

# Set compile options
target_compile_options(minicargo PRIVATE
    -Wall
    -std=c++14
    -g
    -O2
    ${CXXFLAGS_EXTRA}
)

# Set link options
target_link_options(minicargo PRIVATE
    -g
    ${LINKFLAGS_EXTRA}
)

# Include directories
target_include_directories(minicargo PRIVATE
    ${CMAKE_SOURCE_DIR}/tools/common
)

# Link libraries
target_link_libraries(minicargo PRIVATE
    common_lib
    pthread
)

# Set output directory
set_target_properties(minicargo PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/bin"
)

# Platform-specific configurations
if(WIN32)
    # Windows-specific settings
    set_target_properties(minicargo PROPERTIES SUFFIX ".exe")
elseif(APPLE)
    # macOS-specific settings
else()
    # Linux/Unix-specific settings
    find_program(OBJCOPY objcopy)
    find_program(STRIP strip)
    
    if(OBJCOPY AND STRIP)
        add_custom_command(TARGET minicargo POST_BUILD
            COMMAND ${OBJCOPY} --only-keep-debug $<TARGET_FILE:minicargo> $<TARGET_FILE:minicargo>.debug
            COMMAND ${OBJCOPY} --add-gnu-debuglink=$<TARGET_FILE:minicargo>.debug $<TARGET_FILE:minicargo>
            COMMAND ${STRIP} $<TARGET_FILE:minicargo>
        )
    endif()
endif()

# Add dependencies
add_dependencies(minicargo common_lib)

# Generate compile_commands.json for IDE support
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
