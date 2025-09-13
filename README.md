Mitsuba Fork for CMake
===================================

This project is based on [VicentChen/mitsuba](https://github.com/VicentChen/mitsuba), 
a fork of Mitsuba 0.6. The dependency management has been updated to use [vcpkg](https://vcpkg.io/en/) 
instead of the original setup, so you no longer need to worry about dependency or Python version issues.

## Usage

### Environment
The following tools are required:
 - [CMake](https://cmake.org/download/)
 - [vcpkg](https://vcpkg.io/en/)

### Compilation
```bash
mkdir cbuild
cd cbuild
cmake -B . -S .. -DCMAKE_TOOLCHAIN_FILE=[Path to your vcpkg.cmake]
```
This setup has been tested on Visual Studio 2022 and Windows 10.

## Known Issues
Exporting HDR images fails in Debug mode (works fine in Release mode).