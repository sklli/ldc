include(CheckTypeSize)
check_type_size(void* ptr_size)
if(${ptr_size} MATCHES "^4$")
    set(host_model 32)
elseif(${ptr_size} MATCHES "^8$")
    set(host_model 64)
endif()

set(gdb_dflags "")
if((${CMAKE_SYSTEM_NAME} MATCHES "Linux") AND (${LDC_LLVM_VER} GREATER 307))
    execute_process(COMMAND gdb --version
                    COMMAND head -n 1
                    OUTPUT_VARIABLE GDB_VERSION
                    OUTPUT_STRIP_TRAILING_WHITESPACE)
    string(REGEX REPLACE "[^0-9]*([0-9]+[0-9.]*).*" "\\1" GDB_VERSION "${GDB_VERSION}")
    message(STATUS "GDB ${GDB_VERSION} detected")
    if(GDB_VERSION VERSION_LESS "7.6.1")
        set(gdb_flags "NOTLS")
    else()
        set(gdb_flags "ON")
    endif()
    if(GDB_VERSION VERSION_LESS "7.8")
        set(gdb_dflags "-dwarf-version=2")
    endif()
else()
    set(gdb_flags "OFF")
endif()

function(add_testsuite config_name dflags gdbflags model)
    set(name dmd-testsuite${config_name})
    set(outdir ${CMAKE_BINARY_DIR}/${name})

    add_test(NAME clean-${name}
        COMMAND ${CMAKE_COMMAND} -E remove_directory ${outdir})

    set(all_dflags "${dflags} ${gdb_dflags}")

    # The DFLAGS environment variable read by LDMD is used because the DMD
    # testsuite build system provides no way to run the test cases with a
    # given set of flags without trying all combinations of them.
    add_test(NAME ${name}
        COMMAND make -k -C ${PROJECT_SOURCE_DIR}/tests/d2/dmd-testsuite RESULTS_DIR=${outdir} DMD=${LDMD_EXE_FULL} DFLAGS=${all_dflags} MODEL=${model} GDB_FLAGS=${gdbflags} quick
    )
    set_tests_properties(${name} PROPERTIES DEPENDS clean-${name})
endfunction()

string(REGEX REPLACE "[^0-9]*([0-9]+[0-9.]*).*" "\\1" GDB_VERSION "${GDB_VERSION}")
# Would like to specify the "-release" flag for release builds, but some of the
# tests (e.g. 'testdstress') depend on contracts and invariants being active.
# Need a solution integrated with d_do_test.
add_testsuite("-debug" "-g -link-debuglib" "${gdb_flags}" ${host_model})
add_testsuite("" -O3 "OFF" ${host_model})

if(MULTILIB AND host_model EQUAL 64)
    # Also test in 32 bit mode on x86_64 multilib builds.
    add_testsuite("-debug_32" "-g -link-debuglib" "${gdb_flags}" 32)
    add_testsuite("_32" -O3 "OFF" 32)
endif()
