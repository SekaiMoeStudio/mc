#if(",${CMAKE_CURRENT_SOURCE_DIR}," STREQUAL ",${CMAKE_CURRENT_BINARY_DIR},")
if (CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR AND NOT MSVC_IDE)
    # try to clean up (does not work)
    #file(REMOVE "${CMAKE_CURRENT_BINARY_DIR}/CMakeCache.txt")
    message(FATAL_ERROR "ERROR: In-source builds are not allowed, please use an extra build dir.")
endif()

cmake_minimum_required(VERSION 3.10)
project(mrustc CXX)

# Set C++ standard
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_program(CCACHE_FOUND ccache)
if(CCACHE_FOUND)
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
    set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
    message(STATUS "Using ccache: ${CCACHE_FOUND}")
else()
    message(STATUS "Ccache not found. Compiling with cache will be disabled.")
endif(CCACHE_FOUND)

# Find required packages
find_package(ZLIB REQUIRED)

# Configuration options
option(ENABLE_GPROF "Enable gprof profiling" OFF)

# Get Git version information
find_package(Git QUIET)
if(GIT_FOUND)
    execute_process(
        COMMAND ${GIT_EXECUTABLE} show --pretty=%H -s --no-show-signature
        OUTPUT_VARIABLE GIT_FULL_HASH
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    execute_process(
        COMMAND ${GIT_EXECUTABLE} symbolic-ref -q --short HEAD
        OUTPUT_VARIABLE GIT_BRANCH
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
    )
    if(NOT GIT_BRANCH)
        execute_process(
            COMMAND ${GIT_EXECUTABLE} describe --tags --exact-match
            OUTPUT_VARIABLE GIT_BRANCH
            OUTPUT_STRIP_TRAILING_WHITESPACE
            ERROR_QUIET
        )
    endif()
    execute_process(
        COMMAND ${GIT_EXECUTABLE} show -s --pretty=%h --no-show-signature
        OUTPUT_VARIABLE GIT_SHORT_HASH
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    execute_process(
        COMMAND ${GIT_EXECUTABLE} diff-index --quiet HEAD
        RESULT_VARIABLE GIT_IS_DIRTY
    )
endif()

string(TIMESTAMP BUILD_TIME "%a, %d %b %Y %H:%M:%S +0000" UTC)

# Common compiler flags
add_compile_options(
    -Wall
    -Wno-pessimizing-move
    -Wno-misleading-indentation
    -Werror=return-type
    -Werror=switch
    -O2
)

# Include directories
include_directories(
    src/include
    src
    tools/common
)

# Source files grouped by directory
set(MRUSTC_SOURCES
    # Base source files
    src/main.cpp
    src/version.cpp
    src/span.cpp
    src/rc_string.cpp
    src/debug.cpp
    src/ident.cpp

    # AST related files
    src/ast/ast.cpp
    src/ast/types.cpp
    src/ast/crate.cpp
    src/ast/path.cpp
    src/ast/expr.cpp
    src/ast/pattern.cpp
    src/ast/dump.cpp

    # Parse related files
    src/parse/parseerror.cpp
    src/parse/token.cpp
    src/parse/tokentree.cpp
    src/parse/interpolated_fragment.cpp
    src/parse/tokenstream.cpp
    src/parse/lex.cpp
    src/parse/ttstream.cpp
    src/parse/root.cpp
    src/parse/paths.cpp
    src/parse/types.cpp
    src/parse/expr.cpp
    src/parse/pattern.cpp

    # Expand related files
    src/expand/mod.cpp
    src/expand/macro_rules.cpp
    src/expand/cfg.cpp
    src/expand/format_args.cpp
    src/expand/asm.cpp
    src/expand/concat.cpp
    src/expand/stringify.cpp
    src/expand/file_line.cpp
    src/expand/derive.cpp
    src/expand/lang_item.cpp
    src/expand/std_prelude.cpp
    src/expand/crate_tags.cpp
    src/expand/include.cpp
    src/expand/env.cpp
    src/expand/test.cpp
    src/expand/rustc_diagnostics.cpp
    src/expand/proc_macro.cpp
    src/expand/assert.cpp
    src/expand/compile_error.cpp
    src/expand/codegen.cpp
    src/expand/doc.cpp
    src/expand/lints.cpp
    src/expand/misc_attrs.cpp
    src/expand/stability.cpp
    src/expand/panic.cpp
    src/expand/rustc_box.cpp
    src/expand/test_harness.cpp

    # Macro rules
    src/macro_rules/mod.cpp
    src/macro_rules/eval.cpp
    src/macro_rules/parse.cpp

    # Resolve
    src/resolve/use.cpp
    src/resolve/index.cpp
    src/resolve/absolute.cpp
    src/resolve/common.cpp

    # HIR related files
    src/hir/from_ast.cpp
    src/hir/from_ast_expr.cpp
    src/hir/dump.cpp
    src/hir/hir.cpp
    src/hir/hir_ops.cpp
    src/hir/generic_params.cpp
    src/hir/crate_ptr.cpp
    src/hir/expr_ptr.cpp
    src/hir/type.cpp
    src/hir/path.cpp
    src/hir/expr.cpp
    src/hir/pattern.cpp
    src/hir/visitor.cpp
    src/hir/crate_post_load.cpp
    src/hir/inherent_cache.cpp
    src/hir/serialise.cpp
    src/hir/deserialise.cpp
    src/hir/serialise_lowlevel.cpp

    # HIR conversion
    src/hir_conv/expand_type.cpp
    src/hir_conv/constant_evaluation.cpp
    src/hir_conv/resolve_ufcs.cpp
    src/hir_conv/bind.cpp
    src/hir_conv/markings.cpp
    src/hir_conv/lifetime_elision.cpp

    # HIR type checking
    src/hir_typeck/outer.cpp
    src/hir_typeck/common.cpp
    src/hir_typeck/helpers.cpp
    src/hir_typeck/static.cpp
    src/hir_typeck/impl_ref.cpp
    src/hir_typeck/resolve_common.cpp
    src/hir_typeck/expr_visit.cpp
    src/hir_typeck/expr_cs.cpp
    src/hir_typeck/expr_cs__enum.cpp
    src/hir_typeck/expr_check.cpp

    # HIR expansion
    src/hir_expand/annotate_value_usage.cpp
    src/hir_expand/closures.cpp
    src/hir_expand/ufcs_everything.cpp
    src/hir_expand/reborrow.cpp
    src/hir_expand/erased_types.cpp
    src/hir_expand/vtable.cpp
    src/hir_expand/static_borrow_constants.cpp
    src/hir_expand/lifetime_infer.cpp

    # MIR related files
    src/mir/mir.cpp
    src/mir/mir_ptr.cpp
    src/mir/dump.cpp
    src/mir/helpers.cpp
    src/mir/visit_crate_mir.cpp
    src/mir/from_hir.cpp
    src/mir/from_hir_match.cpp
    src/mir/mir_builder.cpp
    src/mir/check.cpp
    src/mir/cleanup.cpp
    src/mir/optimise.cpp
    src/mir/check_full.cpp
    src/mir/borrow_check.cpp

    # Trans related files
    src/trans/trans_list.cpp
    src/trans/mangling_v2.cpp
    src/trans/enumerate.cpp
    src/trans/auto_impls.cpp
    src/trans/monomorphise.cpp
    src/trans/codegen.cpp
    src/trans/codegen_c.cpp
    src/trans/codegen_c_structured.cpp
    src/trans/codegen_mmir.cpp
    src/trans/target.cpp
    src/trans/allocator.cpp
)

# Create common library
add_library(mrustc_lib STATIC
    ${MRUSTC_SOURCES}
)

target_compile_definitions(mrustc_lib PRIVATE
    VERSION_GIT_FULLHASH="${GIT_FULL_HASH}"
    VERSION_GIT_BRANCH="${GIT_BRANCH}"
    VERSION_GIT_SHORTHASH="${GIT_SHORT_HASH}"
    VERSION_BUILDTIME="${BUILD_TIME}"
    VERSION_GIT_ISDIRTY=${GIT_IS_DIRTY}
)

# Build common tools library
add_subdirectory(tools/common)
add_subdirectory(tools/minicargo)

# Create main executable
add_executable(mrustc src/main.cpp)
target_link_libraries(mrustc PRIVATE
    mrustc_lib
    common_lib
    ZLIB::ZLIB
)

# Handle gprof profiling
if(ENABLE_GPROF)
    target_compile_options(mrustc PRIVATE -pg -no-pie)
    target_link_options(mrustc PRIVATE -pg -no-pie)
endif()

# Platform-specific configurations
if(WIN32)
    # Windows-specific settings
    set_target_properties(mrustc PROPERTIES SUFFIX ".exe")
elseif(APPLE)
    # macOS-specific settings
    target_link_options(mrustc PRIVATE -Wl,-all_load)
else()
    # Linux/Unix-specific settings
    find_program(OBJCOPY objcopy)
    find_program(STRIP strip)
    
    if(OBJCOPY AND STRIP)
        add_custom_command(TARGET mrustc POST_BUILD
            COMMAND ${OBJCOPY} --only-keep-debug $<TARGET_FILE:mrustc> $<TARGET_FILE:mrustc>.debug
            COMMAND ${OBJCOPY} --add-gnu-debuglink=$<TARGET_FILE:mrustc>.debug $<TARGET_FILE:mrustc>
            COMMAND ${STRIP} $<TARGET_FILE:mrustc>
        )
    endif()

    # Linux-specific linking options
    target_link_options(mrustc PRIVATE -Wl,--whole-archive -Wl,--no-whole-archive)
endif()

# Installation
install(TARGETS mrustc
    RUNTIME DESTINATION bin
)
