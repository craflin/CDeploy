cmake_minimum_required(VERSION 3.1)
cmake_policy(SET CMP0048 NEW)

option(BUILD_PACKAGE "Build a binary package that provide a Release and Debug build" OFF)

project(example_package VERSION 0.1.0)

include(../../CDeploy)
include(CPack)

if(BUILD_PACKAGE)

    include(ExternalProject)

    ExternalProject_Add(release
        SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}"
        BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/build-release"
        CMAKE_ARGS "-DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/install-release"
        BUILD_COMMAND ${CMAKE_COMMAND} --build "${CMAKE_CURRENT_BINARY_DIR}/build-release" --config Release
        INSTALL_COMMAND ${CMAKE_COMMAND} --build "${CMAKE_CURRENT_BINARY_DIR}/build-release" --config Release --target install
    )

    ExternalProject_Add(debug
        SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}"
        BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/build-debug"
        CMAKE_ARGS "-DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/install-debug"
        BUILD_COMMAND ${CMAKE_COMMAND} --build "${CMAKE_CURRENT_BINARY_DIR}/build-debug" --config Debug
        INSTALL_COMMAND ${CMAKE_COMMAND} --build "${CMAKE_CURRENT_BINARY_DIR}/build-debug" --config Debug --target install
    )

    install(DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/install-debug/" DESTINATION . USE_SOURCE_PERMISSIONS)
    install(DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/install-release/" DESTINATION . USE_SOURCE_PERMISSIONS)

else()

    set(CMAKE_DEBUG_POSTFIX d)

    add_library(mylib STATIC
        src/MyLib.cpp
        include/MyLib.hpp
    )
    target_include_directories(mylib
        PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
        PUBLIC "$<INSTALL_INTERFACE:include>"
    )

    add_executable(mytool
        src/Main.cpp
    )

    install(TARGETS mytool
        DESTINATION bin
        EXPORT ${PROJECT_NAME}Config
    )
    install(TARGETS mylib
        DESTINATION lib
        EXPORT ${PROJECT_NAME}Config
    )
    install(DIRECTORY include
        DESTINATION .
    )
    install(EXPORT ${PROJECT_NAME}Config
        DESTINATION "lib/cmake/${PROJECT_NAME}"
        NAMESPACE example_package::
    )
    include(CMakePackageConfigHelpers)
    write_basic_package_version_file("${PROJECT_NAME}ConfigVersion.cmake" COMPATIBILITY ExactVersion)
    install(FILES "${CMAKE_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake" DESTINATION "lib/cmake/${PROJECT_NAME}")

endif()