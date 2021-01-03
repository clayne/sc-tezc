cmake_minimum_required(VERSION 3.5.1)
project(sc_signal C)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

add_executable(sc_signal signal_example.c sc_signal.h sc_signal.c)

if (NOT CMAKE_C_COMPILER_ID MATCHES "MSVC")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -Wall -pedantic -Werror -D_GNU_SOURCE")
endif ()


# --------------------------------------------------------------------------- #
# --------------------- Test Configuration Start ---------------------------- #
# --------------------------------------------------------------------------- #

include(CTest)
include(CheckCCompilerFlag)

enable_testing()

add_executable(${PROJECT_NAME}_test signal_test.c sc_signal.c)

target_compile_options(${PROJECT_NAME}_test PRIVATE -DSC_SIZE_MAX=1400000ul)

check_c_source_compiles("
    #include <execinfo.h>
    #include <unistd.h>

    int main(int argc, char **argv) {
        void *array[10];
        size_t size = backtrace(array, 10);
        backtrace_symbols_fd(array, size, STDERR_FILENO);
        return 0;
}" HAVE_BACKTRACE)

FIND_LIBRARY(EXECINFO_LIBRARY NAMES execinfo)
IF (EXECINFO_LIBRARY)
    SET(CMAKE_REQUIRED_LIBRARIES "${EXECINFO_LIBRARY}")
ENDIF(EXECINFO_LIBRARY)

if (${HAVE_BACKTRACE})
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DHAVE_BACKTRACE")
endif ()

if (${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
    if ("${CMAKE_C_COMPILER_ID}" STREQUAL "Clang" OR
            "${CMAKE_C_COMPILER_ID}" STREQUAL "GNU")

        target_compile_options(${PROJECT_NAME}_test PRIVATE -fno-omit-frame-pointer)
        target_compile_options(${PROJECT_NAME}_test PRIVATE -DSC_HAVE_WRAP)
        target_compile_options(${PROJECT_NAME}_test PRIVATE -fno-builtin)
    endif ()
endif ()

if ("${CMAKE_C_COMPILER_ID}" STREQUAL "Clang" OR
        "${CMAKE_C_COMPILER_ID}" STREQUAL "AppleClang" OR
        "${CMAKE_C_COMPILER_ID}" STREQUAL "GNU")

    target_compile_options(${PROJECT_NAME}_test PRIVATE -fno-omit-frame-pointer)

    if (SANITIZER)
        target_compile_options(${PROJECT_NAME}_test PRIVATE -fsanitize=${SANITIZER})
        target_link_options(${PROJECT_NAME}_test PRIVATE -fsanitize=${SANITIZER})
    endif ()
endif ()


add_test(NAME ${PROJECT_NAME}_test COMMAND ${PROJECT_NAME}_test)

SET(MEMORYCHECK_COMMAND_OPTIONS
        "-q --log-fd=2 --trace-children=yes --track-origins=yes       \
         --leak-check=full --show-leak-kinds=all  \
         --error-exitcode=255")

add_custom_target(valgrind_${PROJECT_NAME} ${CMAKE_COMMAND}
        -E env CTEST_OUTPUT_ON_FAILURE=1
        ${CMAKE_CTEST_COMMAND} -C $<CONFIG>
        --overwrite MemoryCheckCommandOptions=${MEMORYCHECK_COMMAND_OPTIONS}
        --verbose -T memcheck WORKING_DIRECTORY ${CMAKE_BINARY_DIR})

add_custom_target(check_${PROJECT_NAME} ${CMAKE_COMMAND}
        -E env CTEST_OUTPUT_ON_FAILURE=1
        ${CMAKE_CTEST_COMMAND} -C $<CONFIG> --verbose
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR})

# ----------------------- - Code Coverage Start ----------------------------- #

if (${CMAKE_BUILD_TYPE} MATCHES "Coverage")
    if ("${CMAKE_C_COMPILER_ID}" STREQUAL "GNU")
        target_compile_options(${PROJECT_NAME}_test PRIVATE --coverage)
        target_link_libraries(${PROJECT_NAME}_test gcov)
    else ()
        message(FATAL_ERROR "Only GCC is supported for coverage")
    endif ()
endif ()

add_custom_target(coverage_${PROJECT_NAME})
add_custom_command(
        TARGET coverage_${PROJECT_NAME}
        COMMAND lcov --capture --directory ..
        --output-file coverage.info --rc lcov_branch_coverage=1
        COMMAND lcov --remove coverage.info '/usr/*' '*example*' '*test*'
        --output-file coverage.info --rc lcov_branch_coverage=1
        COMMAND lcov --list coverage.info --rc lcov_branch_coverage=1
)

add_dependencies(coverage_${PROJECT_NAME} check_${PROJECT_NAME})

# -------------------------- Code Coverage End ------------------------------ #


# ----------------------- Test Configuration End ---------------------------- #
