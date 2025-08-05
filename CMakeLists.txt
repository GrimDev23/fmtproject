cmake_minimum_required(VERSION 3.10)

project(fmtproject)

add_executable(fmtproject main.cpp)

find_package(fmt CONFIG REQUIRED)

target_link_libraries(fmtproject 
PRIVATE 
    fmt::fmt
)