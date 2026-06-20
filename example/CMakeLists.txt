# Eggs.Assert
#
# Copyright Agustin K-ballo Berge, Fusion Fenix 2026
#
# Distributed under the Boost Software License, Version 1.0. (See accompanying
# file LICENSE.txt or copy at http://www.boost.org/LICENSE_1_0.txt)

# add_example(<name> <static_lib> <shared_lib>)
#
# Builds <name>.cpp as two executables — one linked against <static_lib> and
# one against <shared_lib> — registered with CTest under the label "example".
function(add_example name static_lib shared_lib)
    foreach(_suffix static shared)
        if(_suffix STREQUAL "static")
            set(_lib ${static_lib})
        else()
            set(_lib ${shared_lib})
        endif()

        set(_target eggs.assert.example.${name}.${_suffix})
        add_executable(${_target} ${name}.cpp)
        target_link_libraries(
            ${_target}
            PRIVATE ${_lib} Eggs::Assert::SuppressAbortDialog
        )
        _eggs_assert_setup_shared(${_target} _eggs_assert_${_suffix})

        if(BUILD_TESTING)
            add_test(
                NAME eggs.assert.example.${name}.${_suffix}
                COMMAND
                    ${CMAKE_COMMAND} -DCMD=$<TARGET_FILE:${_target}>
                    "-DWILL_CRASH=$<CONFIG:Debug>" -P
                    "${PROJECT_SOURCE_DIR}/support/run_verify.cmake"
            )
            set_tests_properties(
                eggs.assert.example.${name}.${_suffix}
                PROPERTIES LABELS "example"
            )
        endif()
    endforeach()
endfunction()

add_example(basic Eggs::Assert::Static Eggs::Assert::Shared)
add_example(
    intercept
    Eggs::Assert::Intercept::Static
    Eggs::Assert::Intercept::Shared
)
