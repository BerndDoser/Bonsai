cmake_minimum_required(VERSION 3.0)

set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++11")

find_package(Eigen3 REQUIRED)
include_directories(${EIGEN3_INCLUDE_DIR})

add_executable(create_tipsy create_tipsy.cpp)
add_executable(read_tipsy read_tipsy.cpp)
add_executable(split_tipsy split_tipsy.cpp)
add_executable(reduce_tipsy reduce_tipsy.cpp)
