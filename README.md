

# CDeploy

CDeploy is a simple CMake centric package manager and package format. It provides the CMake function `deploy_package` to download and unpack a ZIP file and uses [find_package](https://cmake.org/cmake/help/latest/command/find_package.html) in *config mode* to import external pre-build libraries or tools provided by the ZIP file.

There is no global repository that provides packages, but if you are already familiar with modern CMake there is not much you need to learn to create your own CDeploy packages for your project.

## Example

Add the [CDeploy](/CDeploy) file to your CMake project and create a `CMakeLists.txt` file like this:

```cmake
cmake_minimum_required(VERSION 3.1)

project(example_project)

include(CDeploy)

deploy_package(Qt5 5.11.2 "http://my.example-package-repository.com/packages")
deploy_package(libxml2 2.7.8 "http://my.example-package-repository.com/packages")

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

Everything should be packaged in one directory with a unique name. This directory will be the argument for `PATHS` of `find_package`, so it should provide a CMake package configuration with version information.

## Creating a CDeploy Package

CDeploy packages can be created in various ways:

* directly with CMake using exported targets and CPack (which is the preferred way).
* from an external project that is built with or without CMake.
* from an external project with self provided CMake build rules.

Some projects may already produce a CDeploy compatible package that just need to be renamed to match the package naming conventions.

### Directly with CMake/CPack

If you are building your project with CMake and have control over the build rules of the project, you can create a CDeploy package with exported targets and CPack. In Visual Studio builds, it will include a debug and release version of libraries.

Example:

```cmake
cmake_minimum_required(VERSION 3.1)
cmake_policy(SET CMP0048 NEW)

project(example_package VERSION 0.1.0)

include(CDeploy)
include(CPack)

add_executable(mytool
    src/Main.cpp
)
install(TARGETS mytool
    DESTINATION bin
    EXPORT ${PROJECT_NAME}Config
)

add_library(mylib STATIC
    src/MyLib.cpp
    include/MyLib.hpp
)
target_include_directories(mylib
    PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
    PUBLIC "$<INSTALL_INTERFACE:include>"
)
install(TARGETS mylib
    DESTINATION lib
    EXPORT ${PROJECT_NAME}Config
)
install(DIRECTORY include
    DESTINATION .
)

install_deploy_export()
```

The package is then generated using the `package` target in CMake.

```
cmake --build /your/project/dir --target package
```

With Visual Studio, you will have to compile the `DEBUG_BUILD` target first to ensure that a debug is available for packaging. Then build the `package` in `Release` configuration.

```
cmake --build /your/project/dir --target DEBUG_BUILD
cmake --build /your/project/dir --target package --config Release
```

Alternatively, you can configure the project with `-DCDEPLOY_DEBUG_BUILD=ON` to skip the `DEBUG_BUILD` step.

### From an External Project

An external project can be turned into a CDeploy package by compiling it with [ExternalProject_Add](https://cmake.org/cmake/help/latest/module/ExternalProject.html), attaching target importing rules and repackaging it with CPack.

Example:

```cmake
cmake_minimum_required(VERSION 3.1)
cmake_policy(SET CMP0048 NEW)

project(libnstd VERSION 0.1.0)

include(CDeploy)
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
```

If the external project is not built with CMake you will have to customize the `CONFIGURE_COMMAND` and `BUILD_COMMAND` of `ExternalProject_Add`.

The package can be built with:

```
cmake --build /your/project/dir --target package
```

### From an External Project with Self Provided Build Rules

If an external project is not build with CMake, it might be easier to provide your own `CMakeLists.txt` to compile the project with CMake instead of using its build tool chain and repacking its artifacts. [ExternalProject_Add](https://cmake.org/cmake/help/latest/module/ExternalProject.html) can be used to import the sources of the external project.

Example:

```cmake
cmake_minimum_required(VERSION 3.1)
cmake_policy(SET CMP0048 NEW)

project(libfoo VERSION 0.1.0)

include(CDeploy)
include(CPack)
include(ExternalProject)

ExternalProject_Add(sources
    GIT_REPOSITORY "https://github.com/bar/foo.git"
    GIT_SHALLOW True
    GIT_TAG v${PROJECT_VERSION}
    SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/source"
    BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/build"
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
)

set(SOURCES
    "${CMAKE_CURRENT_BINARY_DIR}/source/src/source_file1.cpp"
    "${CMAKE_CURRENT_BINARY_DIR}/source/src/source_file2.cpp"
    # ...
)
set_source_files_properties(${SOURCES}
    PROPERTIES GENERATED True
)

add_library(foo STATIC
    ${SOURCES}
)
add_dependency(foo
    sources
)

target_include_directories(foo
    PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/source/include>"
    PUBLIC "$<INSTALL_INTERFACE:include>"
)
install(TARGETS foo
    DESTINATION lib
    EXPORT ${PROJECT_NAME}Config
)
install(DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/source/include"
    DESTINATION .
)

install_deploy_export()
```

The package is then generated using the `package` target in CMake.

```
cmake --build /your/project/dir --target package
```

With Visual Studio, you will have to compile the `DEBUG_BUILD` target first to ensure that a debug is available for packaging. Then build the `package` in `Release` configuration.

```
cmake --build /your/project/dir --target DEBUG_BUILD
cmake --build /your/project/dir --target package --config Release
```

Alternatively, you can configure the project with `-DCDEPLOY_DEBUG_BUILD=ON` to skip the `DEBUG_BUILD` step.

## Project History

This project is heavily inspired by Daniel Pfeifer's [Effective CMake](https://github.com/boostcon/cppnow_presentations_2017/blob/master/05-19-2017_friday/effective_cmake__daniel_pfeifer__cppnow_05-19-2017.pdf) ([video](https://www.youtube.com/watch?v=bsXLMQ6WgIk)) presentation at [C++Now 2017](https://github.com/boostcon/cppnow_presentations_2017). The package downloading approach is something that I have already used for years, but this is my first attempt at formalizing the package format and providing CMake functions to create them.

## Related Projects

Similar projects are [Hunter](https://hunter.readthedocs.io) and other C++ package managers like [Conan](https://conan.io) and [Vcpkg](https://github.com/microsoft/vcpkg). (Please let me know about other projects that I should add to this list). Hunter seems to be very similar but it relies an globally accessible resources (as far as I understand it) and focuses on pre-build (or integrated) open source software packages. Conan and Vcpkg are external tools that operate on top of CMake and hence will introduce another tool dependency to you build tool set.

## TODO

* Caching of downloaded packages to speed up *clean rebuild builds* in CI.
* Add documentation of `deploy_package`, `deploy_export_dependency`, `deploy_export` and `install_deploy_export`.
* Support CDeploy packages that use `deploy_package` to deploy dependencies.
* Develop a concept to deal with diamond dependency problems with mismatching versions.
