cmake_minimum_required(VERSION 3.10)

project(RubiksCubeProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_library(rubiks_cube_core STATIC
    Models/RubiksCube.cpp
)

target_include_directories(rubiks_cube_core
    PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/Models
)

if(MSVC)
    target_compile_options(rubiks_cube_core PRIVATE /W4)
else()
    target_compile_options(rubiks_cube_core PRIVATE -Wall -Wextra -pedantic)
endif()
