
cmake_minimum_required(VERSION 3.1)
cmake_policy(SET CMP0048 NEW)

project(example_project2 VERSION 0.2.0)

include(ExternalProject)

ExternalProject_Add(release
    SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../example_project2"
    BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/extern-release"
    CMAKE_ARGS "-DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/install-release"
    BUILD_COMMAND ${CMAKE_COMMAND} --build "${CMAKE_CURRENT_BINARY_DIR}/extern-release" --config Release
    INSTALL_COMMAND ${CMAKE_COMMAND} --build "${CMAKE_CURRENT_BINARY_DIR}/extern-release" --config Release --target install
)

ExternalProject_Add(debug
    SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../example_project2"
    BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/extern-debug"
    CMAKE_ARGS "-DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/install-debug"
    BUILD_COMMAND ${CMAKE_COMMAND} --build "${CMAKE_CURRENT_BINARY_DIR}/extern-debug" --config Debug
    INSTALL_COMMAND ${CMAKE_COMMAND} --build "${CMAKE_CURRENT_BINARY_DIR}/extern-debug" --config Debug --target install
)

include(CDeploy)
include(CPack)

install(DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/install-debug/" DESTINATION .)
install(DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/install-release/" DESTINATION .)
