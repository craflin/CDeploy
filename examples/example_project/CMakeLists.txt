cmake_minimum_required(VERSION 3.1)

project(example_project)

include(../../CDeploy)

deploy(example_package 0.1.0 "https://github.com/craflin/CDeploy/releases/download/test")

add_executable(example_binary
    src/Main.cpp
)

target_link_libraries(example_binary
  example_package::mylib
)
