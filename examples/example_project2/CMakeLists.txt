cmake_minimum_required(VERSION 3.1)
cmake_policy(SET CMP0048 NEW)

project(example_project2 VERSION 0.2.0)

include(CDeploy)
include(CPack)

add_library(mylib STATIC
    src/MyLib.cpp
    include/MyLib.hpp
)
target_include_directories(mylib
    PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    PUBLIC $<INSTALL_INTERFACE:include>
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
    DESTINATION lib/${PROJECT_NAME}
    NAMESPACE example_project2::
)
