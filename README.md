

# CDeploy

CDeploy is a simple CMake centric package manager and package format. It provides the CMake function `deploy_package` to download and unpack a ZIP file and uses `find_package` in *config mode* to import external libraries or tools provided by the ZIP file.

## Example

Add the [CDeploy](/CDeploy) file to your CMake project and create a `CMakeLists.txt` file like this:

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

## Creating a CDeploy Package

### Directly with CMake/CPack

If you are building your project with CMake, you can create a CDeploy package with exported targets and CPack. In Visual Studio builds, it will include a debug and release version of libraries.

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
cmake --build /your/projectr/dir --target package
```

With Visual Studio, you will have to compile the `DEBUG_BUILD` target first to ensure that a debug is available for packaging. Then build the `package` in `Release` configuration.

```
cmake --build /your/projectr/dir --target DEBUG_BUILD
cmake --build /your/projectr/dir --target package --config Release
```

Alternatively, you can configure the project with `-DCDEPLOY_DEBUG_BUILD=ON` to skip the `DEBUG_BUILD` step.

### From an External Project

```cmake
todo
```


