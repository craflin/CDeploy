

# CDeploy

CDeploy is a simple CMake centric package manager and package format. It provides the CMake function `deploy_package` to download and unpack a ZIP file and uses `find_package` in *config mode* to import external libraries or tools provided by the ZIP file.

## Example

Add the [CDeploy](/CDeploy) to your CMake project and create a `CMakeLists.txt` file like this:

```cmake
cmake_minimum_required(VERSION 3.1)

project(example_project)

include(CDeploy)

deploy_package(Qt5 5.11.2 "http://my.example-package-repostory.com/packages")
deploy_package(libxml2 2.7.8 "http://my.example-package-repostory.com/packages")

add_executable(example_binary
    Main.cpp
)

target_link_libraries(example_binary
  Qt5::Core libxml2
)
```

## Package Format

### Package Name

&lt;name&gt;-&lt;version&gt;-&lt;os&gt;-&lt;arch&gt;-&lt;compiler&gt;.zip

* &lt;name&gt; is the name of the imported product
* &lt;version&gt; is a generic version string
* &lt;os&gt; is the target operating system or distribution name and version
* &lt;arch&gt; is the target architecture (x86, x64, ppc64, etc.)
* &lt;compiler&gt; is a shorted name of the compiler with version number (gcc7.2, vs2015, etc.)

Example:

libxml2-2.7.8-ubuntu16.04-x64-gcc7.2.zip

### Package Contents

Everything should be packaged in one directory with a unique name. This directory will be the argument for `PATHS` of `find_package`, so it should provide a CMake package configuration.

## Creating a CMake Deploy Package

### Directly with CMake/CPack

If you are building your project with CMake, you can create a CDeploy package using CPack.

```cmake
cmake_minimum_required(VERSION 3.1)
cmake_policy(SET CMP0048 NEW)

project(example_package VERSION 0.2.0)

include(CDeploy)
include(CPack)

set(CMAKE_DEBUG_POSTFIX d)

add_library(mylib STATIC
    src/MyLib.cpp
    include/MyLib.hpp
)
target_include_directories(mylib
    PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>;
    PUBLIC $<INSTALL_INTERFACE:include>;
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
    NAMESPACE example_package::
)
include(CMakePackageConfigHelpers)
write_basic_package_version_file("${PROJECT_NAME}ConfigVersion.cmake" COMPATIBILITY ExactVersion)
install(FILES "${CMAKE_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake" DESTINATION "lib/cmake/${PROJECT_NAME}")
```

The package is then generated using the `package` target in CMake.

If you want to create a package with multiple configurations (e.g. Release and Debug) you will need a to build both configurations individually. This can be done with a CMake project dedicated to building the package or using CMake option and `ExternalProject_Add`:

```cmake
cmake_minimum_required(VERSION 3.1)
cmake_policy(SET CMP0048 NEW)

option(BUILD_PACKAGE "Build a binary package that provide a Release and Debug build" OFF)

project(example_package VERSION 0.2.0)

include(CDeploy)
include(CPack)

if(BUILD_PACKAGE)

    include(ExternalProject)

    ExternalProject_Add(release
        SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}"
        BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/extern-release"
        CMAKE_ARGS "-DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/install-release"
        BUILD_COMMAND ${CMAKE_COMMAND} --build "${CMAKE_CURRENT_BINARY_DIR}/extern-release" --config Release
        INSTALL_COMMAND ${CMAKE_COMMAND} --build "${CMAKE_CURRENT_BINARY_DIR}/extern-release" --config Release --target install
    )

    ExternalProject_Add(debug
        SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}"
        BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/extern-debug"
        CMAKE_ARGS "-DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/install-debug"
        BUILD_COMMAND ${CMAKE_COMMAND} --build "${CMAKE_CURRENT_BINARY_DIR}/extern-debug" --config Debug
        INSTALL_COMMAND ${CMAKE_COMMAND} --build "${CMAKE_CURRENT_BINARY_DIR}/extern-debug" --config Debug --target install
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
```

### From an External Project

```cmake
todo
```


