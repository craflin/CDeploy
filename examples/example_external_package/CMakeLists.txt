cmake_minimum_required(VERSION 3.1)
cmake_policy(SET CMP0048 NEW)

project(libnstd VERSION 0.1.0)

include(../../CDeploy)
include(CPack)
include(ExternalProject)

if(MSVC)
    ExternalProject_Add(libnstd-debug
        GIT_REPOSITORY "https://github.com/craflin/libnstd.git"
        GIT_SHALLOW True
        SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/source"
        BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/build-debug"
        BUILD_COMMAND ${CMAKE_COMMAND} --build "${CMAKE_CURRENT_BINARY_DIR}/build-debug" --config Debug
        INSTALL_COMMAND ""
    )
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/build-debug/src/Debug/libnstd.lib"
        DESTINATION lib
        RENAME libnstdd.lib
    )
    deploy_export(libnstd LIBRARY STATIC
        CONFIGURATION Debug
        IMPORTED_LOCATION "lib/libnstdd.lib"
        INTERFACE_INCLUDE_DIRECTORIES "include"
    )
endif()

ExternalProject_Add(libnstd-release
    GIT_REPOSITORY "https://github.com/craflin/libnstd.git"
    GIT_SHALLOW True
    SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/source"
    BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/build-release"
    BUILD_COMMAND ${CMAKE_COMMAND} --build "${CMAKE_CURRENT_BINARY_DIR}/build-release" --config Release
    INSTALL_COMMAND ${CMAKE_COMMAND} -E echo Skipped install
)
install(DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/source/include" DESTINATION .)

if(MSVC)
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/build-release/src/Release/libnstd.lib"
        DESTINATION lib
    )
    deploy_export(libnstd LIBRARY STATIC
        CONFIGURATION Release
        IMPORTED_LOCATION "lib/libnstd.lib"
        INTERFACE_INCLUDE_DIRECTORIES "include"
    )
else()
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/build-release/src/libnstd.a"
        DESTINATION lib
    )
    deploy_export_dependency(Threads)
    deploy_export(libnstd LIBRARY STATIC
        IMPORTED_LOCATION "lib/libnstd.a"
        INTERFACE_INCLUDE_DIRECTORIES "include"
        PROPERTIES
            INTERFACE_LINK_LIBRARIES Threads::Threads
    )
endif()

install_deploy_export()
