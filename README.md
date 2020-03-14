

# CDeploy

CDeploy is a simple CMake centric package format and package manager. It uses a simple CMake function (`deploy`) to download and unpack a ZIP or TAR file and uses `find_package` in *module mode* to import some CMake targets.

## Example

```cmake
cmake_minimum_required(VERSION 3.1)

project(example_project)

include(CDeploy)

deploy(Qt5 5.11.2 http://my.example-package-repostory.com/packages)
deploy(libxml2 2.7.8 http://my.example-package-repostory.com/packages)

add_executable(example_binary
    Main.cpp
)

target_link_libraries(example_binary
  Qt5::Core libxml2
)
```

## Package Format

### Package Name

<name>-<version>-<os>-<arch>-<compiler>.zip

* <name> is the name of the imported product
* <version> is a generic version string
* <os> is operating system or distribution name and version
* <arch> is x86, x64, ppc64, etc.
* <compiler> is a shorted name of the compiler with version number

Example:

libxml2-2.7.8-ubuntu16.04-x64-gcc7.2.0.zip

### Package Contents

Everything should be packaged in one directory with a unique name. This directory will be the argument `PATHS` for `find_package`, so it should provide a CMake package configuration.

## Creating a CMake Deploy Package

### Directly with CPack

If you are building your project with CMake, you can create a CDeploy package using CPack.

```cmake
cmake_minimum_required(VERSION 3.1)
cmake_policy(SET CMP0048 NEW)

project(example_project2 VERSION 0.2.0)

include(CDeploy)
include(CPack)

set(CMAKE_DEBUG_POSTFIX d)

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
```

The package is then generated using the `package` target in CMake.

If you want to create a package with multiple configurations (e.g. Release and Debug) you will need a CMake project dedicated to build the package:

```cmake
todo
```

### From an External Project

```cmake
todo
```


