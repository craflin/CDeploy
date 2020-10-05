

# CDeploy

CDeploy is a simple CMake centric package manager and package format. It provides the CMake function `deploy_package` to download and unpack a ZIP file and uses [find_package](https://cmake.org/cmake/help/latest/command/find_package.html) in *config mode* to import external pre-build libraries, tools or other resources provided by the downloaded file.

There is no global repository for packages, but if you are already familiar with modern CMake there is not much you need to learn to create your own CDeploy packages for your project. The idea is that each project maintains its own file server for dependencies unless they can be downloaded directly from another projects that provide CDeploy compatible packages.

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

`<name>-<version>[-<os>][-<arch>][-<compiler>].zip`

* `<name>` is the name in lower case of the imported product
* `<version>` is a generic version string
* `<os>` is the target operating system or distribution name and version
* `<arch>` is the target architecture (x86, x64, ppc64, etc.)
* `<compiler>` is a shorted name of the compiler with version number (gcc7, vs2015, etc.)

Example:

libxml2-2.7.8-ubuntu18.04-x64-gcc7.zip

### Package Contents

Everything should be packaged in one directory with a unique name. This directory will be the argument for `PATHS` of `find_package`, so it should provide a CMake package configuration with version information.

## Creating a CDeploy Package

CDeploy packages can be created in various ways:

* directly with CMake using exported targets and CPack.
* from an external project that is built with or without CMake.
* from an external project and your own CMake build rules.

Some projects may already produce a CDeploy compatible package that just need to be renamed to match the package naming conventions.

### Directly with CMake/CPack

If you are building your project with CMake and have control over the build rules (`CMakeLists.txt` files) of the project, you can create a CDeploy package with CPack by using [install(TARGET <name> ...)](https://cmake.org/cmake/help/latest/command/install.html) with the `EXPORT` option set to `${PROJECT_NAME}Config`. In Visual Studio builds, it will include a debug and release version of libraries.

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

With Visual Studio, you will have to compile the target `DEBUG_BUILD` first to ensure that a debug is available for packaging. Then build the `package` in `Release` configuration.

```
cmake --build /your/project/dir --target DEBUG_BUILD
cmake --build /your/project/dir --target package --config Release
```

Alternatively, you can configure the project with `-DCDEPLOY_DEBUG_BUILD=ON` to skip the `DEBUG_BUILD` step.

### From an External Project

An external project can be turned into a CDeploy package by compiling it with [ExternalProject_Add](https://cmake.org/cmake/help/latest/module/ExternalProject.html), exporting its targets with `deploy_export` and repackaging it with CPack.

Example:

```cmake
cmake_minimum_required(VERSION 3.1)
cmake_policy(SET CMP0048 NEW)

project(libnstd VERSION 0.1.0)

set(CDEPLOY_NO_DEBUG_BUILD True)

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
    INSTALL_COMMAND ""
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

### From an External Project with Your Own CMake Build Rules

If an external project is not build with CMake, it might be easier to provide your own `CMakeLists.txt` file to compile the project with CMake instead of using its native build tool chain and repacking its artifacts. [ExternalProject_Add](https://cmake.org/cmake/help/latest/module/ExternalProject.html) can be used to import the sources of the external project. 

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
add_dependencies(foo
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

With Visual Studio, you will have to compile the target `DEBUG_BUILD` first to ensure that a debug is available for packaging. Then build the `package` in `Release` configuration.

```
cmake --build /your/project/dir --target DEBUG_BUILD
cmake --build /your/project/dir --target package --config Release
```

Alternatively, you can configure the project with `-DCDEPLOY_DEBUG_BUILD=ON` to skip the `DEBUG_BUILD` step.

## CDeploy Functions

### When Using CDeploy Packages

#### `deploy_package`

```
deploy_package(<package_name> <version> [<repository_url>]
    [NO_OS] [NO_ARCH] [NO_COMPILER] [NO_CACHE] [NO_CONFIG_MAPPING] [COMPONENTS components...])
```

The function downloads and unpacks a package file from the HTTP folder `<repository_url>` or an URL defined with `CDEPLOY_REPOSITORY` into the binary folder (`CMAKE_BINARY_DIR`) and uses CMake's [find_package](https://cmake.org/cmake/help/latest/command/find_package.html) in *config mode* to import the package. The optional `COMPONENTS` parameter is passed on to `find_package`.

The package has to be named `<package_name>-<version>[-<target_os>][-<target_arch>][-<target_compiler>].zip` where `<target_os>`, `<target_arch>` and `<target_compiler>` can be omitted from the package file name using the options `NO_OS`, `NO_ARCH` and `NO_COMPILER`. `<target_os>`, `<target_arch>` and `<target_compiler>` are automatically detected in the local build environment.

If `NO_CACHE` is set, a locally available package (in the exact version) or an already downloaded package will not be considered and the package will always be download from the repository.

If `NO_CONFIG_MAPPING` is set, [MAP_IMPORTED_CONFIG_<CONFIG>](https://cmake.org/cmake/help/latest/prop_tgt/MAP_IMPORTED_CONFIG_CONFIG.html) will not be set to map all project configurations other than *Debug* to the *Release* version of the libraries and tools provided by the deployed package. This does only affect Visual Studio builds.

### When Creating CDeploy Packages

#### `deploy_export`

```
deploy_export(<name> (INTERFACE | LIBRARY STATIC | LIBRARY SHARED | EXECUTABLE)
    [IMPORTED_LOCATION <path>]
    [CONFIGURATION (Release | Debug)]
    [IMPORTED_IMPLIB <path>]
    [INTERFACE_INCLUDE_DIRECTORIES <dir> ...]
    [INTERFACE_SOURCES <source> ...]
    [INTERFACE_COMPILE_DEFINITIONS <definition> ...]
    [INTERFACE_COMPILE_FEATURES <feature> ...]
    [INTERFACE_COMPILE_OPTIONS <option> ...]
    [INTERFACE_LINK_LIBRARIES <library> ...]
    [PROPERTIES <name> <value> ... ])
```

The function declares that an interface library (`INTERFACE`), static library (`LIBRARY STATIC`), shared library (`LIBRARY SHARED`) or executable (`EXECUTABLE`) can be found in the generated package.

* `IMPORTED_LOCATION` sets the relative location of the static library, shared library or executable in the generated package.
* `CONFIGURATION` can be used in Visual Studio builds with `CDEPLOY_NO_DEBUG_BUILD` option to declare if the exported library or executable is the Debug or Release version.
* `IMPORTED_IMPLIB` sets the relative location in the generated package to a import library of a shared library.
* `INTERFACE_INCLUDE_DIRECTORIES` sets the relative location to include directories of a interface, static or shared library in the generated package.
* `INTERFACE_SOURCES` sets the relative location to interface source files of an imported interface library in the generated package.
* `INTERFACE_COMPILE_DEFINITIONS` sets the `INTERFACE_COMPILE_DEFINITIONS` property of the imported target.
* `INTERFACE_COMPILE_FEATURES` sets the `INTERFACE_COMPILE_FEATURES` property of the imported target.
* `INTERFACE_COMPILE_OPTIONS` sets the `INTERFACE_COMPILE_OPTIONS` property of the imported target.
* `INTERFACE_LINK_LIBRARIES` sets the `INTERFACE_LINK_LIBRARIES` property of the imported target.
* `PROPERTIES` sets additional properties  of a imported library when it is deployed.

#### `deploy_export_dependency`

`deploy_export_dependency(<name> <flags>)`

The function declares a dependency of the generated package that uses `deploy_export` . This dependency should be resolvable by using `find_dependency(<name> <flags>)` when the package is deployed.

#### `install_deploy_export`

```
install_deploy_export()
```

The function adds the export of installed targets or targets exported with `deploy_export` to the generated package.

## CDeploy Options

These options can be set before `include(CDeploy)` to configure the package creation or package downloading.

* `CDEPLOY_NO_DEBUG_BUILD` - Disable the debug build generation when creating a package with Visual Studio. This should be set if a debug build is not required or if the debug build is already included in the package with other build rules.
* `CDEPLOY_NO_OS` - Do not include the operating name in the package name. This is usually combined with `CDEPLOY_NO_ARCH` and `CDEPLOY_NO_COMPILER` to create an interface library package or a package that provides some other resources that can be used independently of the operating system. Such a package can be deployed with `deploy_package` using the `NO_OS`, `NO_ARCH` and `NO_COMPILER` options.
* `CDEPLOY_NO_ARCH` - Do not include the package architecture in the package name. Such a package can be deployed with `deploy_package` using the `NO_ARCH` option.
* `CDEPLOY_NO_COMPILER` - Do not include package compiler name and version in the package name. Such a package can be deployed with `deploy_package` using the `NO_COMPILER` option.
* `CDEPLOY_CACHE_DIR` - Store downloaded packages in the given directory. The cache directory can also be set using an environment variable with the same name. The default value is `~/.cmake/downloadcache`.
* `CDEPLOY_REPOSITORY` - The repository URL to be used when it was left blank in the `deploy_package` command. It can also be set using an environment variable with the same name.

## Dealing with Diamond Dependency Problems

The diamond dependency problems occurs if you have a dependency that depends on a package in a different version than another direct or indirect dependency on the same package. This can be avoided by not having dependencies in a package that rely on an exact version of a package. So, CDeploy packages should not use `deploy_package` to deploy dependencies but can use [find_dependency](https://cmake.org/cmake/help/latest/module/CMakeFindDependencyMacro.html) to ensure that a dependency is available. The dependencies should then be deployed by the the top level `CMakeLists.txt` file, which ensures that all packages use the same version of a dependency. This approach works as long as the dependency that is shared by multiple packages does not break its downwards compatibility. 

## Project History

This project is heavily inspired by Daniel Pfeifer's [Effective CMake](https://github.com/boostcon/cppnow_presentations_2017/blob/master/05-19-2017_friday/effective_cmake__daniel_pfeifer__cppnow_05-19-2017.pdf) ([video](https://www.youtube.com/watch?v=bsXLMQ6WgIk)) presentation at [C++Now 2017](https://github.com/boostcon/cppnow_presentations_2017). The package downloading approach is something that I have already used for years, but this is my first attempt at formalizing the package format and providing CMake functions to create them.

## Related Projects

Similar projects are [Hunter](https://hunter.readthedocs.io) and other C++ package managers like [Conan](https://conan.io) and [Vcpkg](https://github.com/microsoft/vcpkg). (Please let me know about other projects that I should add to this list). Hunter seems to be very similar but it relies an globally accessible resources (as far as I understand it) and focuses on pre-build (or integrated) open source software packages. Conan and Vcpkg are external tools that operate on top of CMake and hence will introduce another tool dependency to you build tool set.
