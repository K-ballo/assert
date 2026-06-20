# Eggs.Assert
#
# Copyright Agustin K-ballo Berge, Fusion Fenix 2026
#
# Distributed under the Boost Software License, Version 1.0. (See accompanying
# file LICENSE.txt or copy at http://www.boost.org/LICENSE_1_0.txt)

# ── Verify interface header sets ────────────────────────────────────────────

foreach(_suffix static shared)
    add_test(
        NAME eggs.assert.verify_headers.${_suffix}
        COMMAND
            ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --target
            _eggs_assert_${_suffix}_verify_interface_header_sets --config
            $<CONFIG>
    )
    set_tests_properties(
        eggs.assert.verify_headers.${_suffix}
        PROPERTIES LABELS "header" RESOURCE_LOCK cmake_build
    )
endforeach()

# add_lib_test(<name> [WILL_CRASH <expr>] [DEBUG_ONLY])
#
# Builds <name>.cpp as two test executables — one linked against
# _eggs_assert_static and one against _eggs_assert_shared — and registers
# both with CTest under the label "unit" with names
# eggs.assert.<name>.static / eggs.assert.<name>.shared.
#
#   WILL_CRASH <expr>   generator expression; truthy = expects abort,
#                       falsy/omitted = expects clean exit (code 0).
#
#   DEBUG_ONLY          the source #errors when NDEBUG is defined, so this
#                       flag compiles a no-op stub in non-Debug configs and
#                       restricts CTest registration to Debug only. On
#                       single-config generators the target is skipped
#                       entirely when CMAKE_BUILD_TYPE is not Debug.
function(add_lib_test name)
    cmake_parse_arguments(PARSE_ARGV 1 ARG "DEBUG_ONLY" "WILL_CRASH" "")

    # For single-config generators, skip Debug-only tests entirely when not Debug.
    if(
        ARG_DEBUG_ONLY
        AND NOT CMAKE_CONFIGURATION_TYPES
        AND NOT CMAKE_BUILD_TYPE STREQUAL "Debug"
    )
        return()
    endif()

    foreach(_suffix static shared)
        set(_target ${name}_${_suffix})
        add_executable(${_target})
        if(ARG_DEBUG_ONLY)
            # Non-Debug configs compile a no-op stub so the source (which
            # #errors on NDEBUG) is never compiled in Release. CTest is gated
            # via CONFIGURATIONS.
            target_sources(
                ${_target}
                PRIVATE
                    "$<$<CONFIG:Debug>:${CMAKE_CURRENT_SOURCE_DIR}/${name}.cpp>"
                    "$<$<NOT:$<CONFIG:Debug>>:${CMAKE_CURRENT_FUNCTION_LIST_DIR}/empty_stub.cpp>"
            )
            set(_configurations CONFIGURATIONS Debug)
        else()
            target_sources(${_target} PRIVATE "${name}.cpp")
            unset(_configurations)
        endif()
        target_link_libraries(
            ${_target}
            PRIVATE _eggs_assert_${_suffix} Eggs::Assert::SuppressAbortDialog
        )

        _eggs_assert_setup_shared(${_target} _eggs_assert_${_suffix})
        add_test(
            NAME eggs.assert.${name}.${_suffix}
            ${_configurations}
            COMMAND
                ${CMAKE_COMMAND} -DCMD=$<TARGET_FILE:${_target}>
                -DWILL_CRASH=${ARG_WILL_CRASH} -P
                "${PROJECT_SOURCE_DIR}/support/run_verify.cmake"
        )
        set_tests_properties(
            eggs.assert.${name}.${_suffix}
            PROPERTIES LABELS "unit"
        )
    endforeach()
endfunction()

# ── Re-includability ──────────────────────────────────────────────────────────

add_lib_test(reinclusion_h_disable)
add_lib_test(reinclusion_h_enable WILL_CRASH 1)

add_lib_test(reinclusion_hpp_disable)
add_lib_test(reinclusion_hpp_enable WILL_CRASH 1)

# ── Pass / fail ───────────────────────────────────────────────────────────────

add_lib_test(assert_pass)
add_lib_test(assert_hpp_fail WILL_CRASH "$<CONFIG:Debug>")
add_lib_test(assert_h_fail WILL_CRASH "$<CONFIG:Debug>")

# ── Comma in expression ───────────────────────────────────────────────────────

add_lib_test(assert_comma)

# ── Single evaluation ─────────────────────────────────────────────────────────

add_lib_test(assert_single_eval DEBUG_ONLY)

# ── assert.h in C mode ────────────────────────────────────────────────────────

foreach(_suffix static shared)
    add_executable(assert_h_c_${_suffix} assert_h_c.c)
    set_target_properties(assert_h_c_${_suffix} PROPERTIES LINKER_LANGUAGE CXX)
    target_link_libraries(
        assert_h_c_${_suffix}
        PRIVATE _eggs_assert_${_suffix} Eggs::Assert::SuppressAbortDialog
    )

    _eggs_assert_setup_shared(assert_h_c_${_suffix} _eggs_assert_${_suffix})

    add_test(
        NAME eggs.assert.assert_h_c.${_suffix}
        COMMAND
            ${CMAKE_COMMAND} -DCMD=$<TARGET_FILE:assert_h_c_${_suffix}>
            -DWILL_CRASH=$<CONFIG:Debug> -P
            "${PROJECT_SOURCE_DIR}/support/run_verify.cmake"
    )
    set_tests_properties(
        eggs.assert.assert_h_c.${_suffix}
        PROPERTIES LABELS "unit"
    )
endforeach()
