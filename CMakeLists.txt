cmake_minimum_required(VERSION 3.20)

project(chess_engine LANGUAGES CXX)

option(CHESS_ENABLE_SANITIZERS "Enable AddressSanitizer and UndefinedBehaviorSanitizer" OFF)

add_library(chess_core INTERFACE)
target_compile_features(chess_core INTERFACE cxx_std_20)

function(chess_enable_warnings target)
    if(MSVC)
        target_compile_options(${target} PRIVATE /W4 /permissive- /Zc:__cplusplus)
    elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
        target_compile_options(${target} PRIVATE -Wall -Wextra -Wpedantic)
    endif()
endfunction()

function(chess_enable_sanitizers target)
    if(NOT CHESS_ENABLE_SANITIZERS)
        return()
    endif()

    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
        target_compile_options(${target} PRIVATE -fsanitize=address,undefined -fno-omit-frame-pointer)
        target_link_options(${target} PRIVATE -fsanitize=address,undefined)
    else()
        message(WARNING "CHESS_ENABLE_SANITIZERS is unsupported by ${CMAKE_CXX_COMPILER_ID}")
    endif()
endfunction()

add_executable(chess_engine src/main.cpp)
target_link_libraries(chess_engine PRIVATE chess_core)
chess_enable_warnings(chess_engine)
chess_enable_sanitizers(chess_engine)

include(CTest)

if(BUILD_TESTING)
    add_executable(chess_tests tests/smoke_test.cpp)
    target_link_libraries(chess_tests PRIVATE chess_core)
    chess_enable_warnings(chess_tests)
    chess_enable_sanitizers(chess_tests)

    add_test(NAME chess.smoke COMMAND chess_tests)
endif()
