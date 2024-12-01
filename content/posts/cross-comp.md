+++
date = '2024-12-01T00:26:51+01:00'
draft = false
title = 'The art of chroot-less cross-compilation'
+++

This article was written for the [PortMaster community](https://portmaster.games/), which ports various pieces of software to ARM-based handheld devices. If it can help anyone else, that's great too!

## Introduction

Compiling C and C++ programs for ARM architectures from a non-ARM device usually involve creating a new filesystem based on the target architecture, and then changing the current root directory to this filesystem using `chroot`. Then, the host uses build tools from this new root through an emulation layer, for instance qemu, to build native binaries for the target architecture. With the architecture difference between the host and the target abstracted away by emulation or virtualization, build commands are the exact same than for a native build from the host, but generates binaries for the target architecture.[^1]

However, emulation has a non-negligeable cost in terms of compute resource usage, significantly slowing down the CPU-intensive build processes. That is where cross-compilation comes in, where we build software for a target architecture different than the one of the host running the compiler, without emulation. Some tools for embedded development, such as [buildroot](https://buildroot.org/), specialize in making cross-compilation toolchains easier to use. buildroot can generate a complete bootable Linux environment and cross-compilation toolchain, but is unnecessarily complex to set up to build only a few executables. Many Linux distributions provide pre-compiled cross-compilation toolchains and utilities, such as Debian (in recent releases), which we will use today. Debian provides most of the tools one could need to cross-build simple programs for ARM architectures[^2], without the hassle and slowness of setting up a full root filesystem for the target architecture, but at the cost of some toolchain gymnastics.

From now on, we assume that we run commands on a fresh Debian Bookworm install on a x86_64 machine, for example from a docker container:

```bash
docker run -it debian:bookworm
```

This post also assumes basic knowledge of C/C++ programming, Linux and compilation tools.

## Compiling a first program

By design, the `gcc` compiler only targets a single computer architecture, which means that we need a different compiler for each target architecture. The `gcc -v` command can print the target architecture. In our Debian container, it is `Target: x86_64-pc-linux-gnu`.

Let's install the gcc compiler targetting ARM64:

```bash
apt install gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu linux-libc-dev-arm64-cross
```

We now have a new gcc executable build to target the `aarch64` architecture.

```bash
aarch64-linux-gnu-gcc -v
```

displays

```bash
Target: aarch64-linux-gnu
```

as expected[^3]. We now have a cross-compiler! Let's create a sample C `main.c` program and compile it for ARM64:

```c
#include <stdio.h>

int main() {
  printf("Cross builds are fun!");
  return 0;
}
```

After compiling it with `aarch64-linux-gnu-gcc main.c -o mainc.aarch64`, we have an ARM executable. The `file` commands confirms that:

```
main.aarch64: ELF 64-bit LSB pie executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, BuildID[sha1]=6c176c663804f2481a7973a7a2fcdabe30bd3ac8, for GNU/Linux 3.7.0, not stripped
```

Note that you cannot run this program, as we compile for aarch64 but we do not emulate the architecture, unlike qemu-based solution. Running it will error out:

```bash
$ ./mainc.aarch64
/mainc.aarch64: cannot execute binary file: Exec format error`.
```

Compiling C++ code works the same way. Install the Debian packages `g++-aarch64-linux-gnu libstdc++-11-dev-arm64-cross`, and create a `main.cxx` file with the following content:

```cpp
#include <iostream>

int main()
{
  std::cout << "Cross builds are fun!" << std::endl;
  return 0;
}
```

You can compile it with the command

```bash
aarch64-linux-gnu-g++ main.cxx -o maincpp.aarch64
```

Now that you have the basics, let's move to builds using third-party libraries.

## Working with multiarch libraries

Real-world programs often rely on external libraries that we need to link our program against. There are several ways we can tackle that problem in a cross-compilation environment; the first and most simple of which is using the libraries provided directly by the Linux distribution. Debian has "multiarch"[^4], which lets the user install libraries from several architectures on the same machine. To use that, we need to enable the architectures we want to target, using the command

```bash
dpkg --add-architecture arm64 && apt update
```

Let's assume that we want to compile a C++ program using the SDL2 library, that displays the number of currently connected joysticks:

```cpp
#include <iostream>
#include "SDL.h"

int main()
{
  SDL_InitSubSystem(SDL_INIT_JOYSTICK);
  int numControllers = SDL_NumJoysticks();
  std::cout << "Found " << numControllers << " joysticks." << std::endl;
  return 0;
}
```

Compiling this program using `aarch64-linux-gnu-g++` as we did before will fail, because `gcc` cannot find the SDL2 include anywhere in the default configuration paths.

We need to install the ARM64 version of SDL2 we can link our program against: `apt install libsdl2-dev:arm64 --no-install-recommends -y`. To compile our program, we need to find both the header `SDL2.h` and the shared library file `libSDL2.so`. Where did Debian install them? You can find out with the command `dpkg -L libsdl2-dev:arm64`:

```
[...]
/usr/include/SDL2/SDL.h
[...]
/usr/lib/aarch64-linux-gnu/libSDL2.so
```

Note that the `/usr/lib/aarch64-linux-gnu/` directory contains aarch64 libraries installed by Debian multiarch packages. `/usr/aarch64-linux-gnu/` also contains libraries and executables that you might need during the build process. Now, compile our program and link it against SDL2, specifying these two paths:

```bash
aarch64-linux-gnu-g++ controllers.cxx -o controllers.aarch64 -I/usr/include/SDL2 -L/usr/lib/aarch64-linux-gnu/libSDL2.so -lSDL2
```

If the program you are compiling happens to depend on a library that Debian does not package (or if the version in the repositories is not the one you need), you will need to build it from source before you can link against it.

## CMake integration

CMake is a popular build system generator used by many projects that help streamline the build process and facilitate the use of third-party libraries.

We first create a `CMakeLists.txt` file describing how to build the previous program:

```cmake
cmake_minimum_required(VERSION 3.18)
project(controllers LANGUAGES CXX)
find_package(SDL2 REQUIRED)

add_executable(controllers controllers.cxx)
target_link_libraries(controllers PRIVATE ${SDL2_LIBRARIES})
target_include_directories(controllers PRIVATE ${SDL2_INCLUDE_DIRS})
```

CMake supports cross-compilation via the use of "toolchains", which are in practice `.cmake` files describing the target platform: which compiler to use, where to find libraries, etc [^5]. A toolchain for our current system building programs for aarch64 could be something like this:


```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)

set(CMAKE_FIND_ROOT_PATH  /usr/lib/aarch64-linux-gnu/)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
```

The first lines are pretty straightforward: we set target system and architecture, and the compilers that are to be used for the build. Then, we specify the path that should be used when CMake is looking for files. In this case, we set it to the directory where Debian puts aarch64 libraries. In some cases, you might need to add more paths to this variable for CMake to find some packages correctly. The last 3 lines tell CMake that when looking for programs to be executed as part of the build process, it should *not* use the root path we just set, but that when looking for libraries, it should *only* look in the defined path: libraries found outside of `/usr/lib/aarch64-linux-gnu` would not be `aarch64` libraries compatible with the executable we are building.

After putting this toolchain in a file `toolchain-gcc-aarch64-deb.cmake`, you can use it as a CMake variable from your source tree (install packages `cmake` & `ninja-build` beforehand):

```bash
cmake -S. -Bbuild -DCMAKE_TOOLCHAIN_FILE=toolchain-gcc-aarch64-deb.cmake -GNinja
cmake --build ./build
```

A toolchain is specific to a build environment and to a target architecture, so you may need multiple toolchains if you target multiple architectures.

Most problems you will be facing when cross-compiling with CMake will probably be related to CMake not finding the packages/libraries/files you need. A useful option to help debugging is `--debug-find`, which traces the locations where the files are searched for, and where it found them. Read carefully the documentation for `find_X` commands and you will eventually figure it out. 

## What about clang?

Clang/LLVM is another compiler toolchain that, unlike `gcc`, is a native cross-compiler [^6]. That means the same compiler executable can target multiple architectures. To use it to build aarch64 binaries, we use the `--target` option with the ARM64 triple:

```bash
clang main.c -o main_clang.aarch64 --target=aarch64-linux-gnu
clang++ main.cxx -o maincpp_clang.aarch64 --target=aarch64-linux-gnu
```

Clang can also be used in combination with CMake. Edit the variables setting up compilers in the toolchain file:

```cmake
set(CMAKE_C_COMPILER clang)
set(CMAKE_CXX_COMPILER clang++)
set(CMAKE_C_COMPILER_TARGET aarch64-linux-gnu)
set(CMAKE_CXX_COMPILER_TARGET aarch64-linux-gnu)
```

[^1]: PortMaster's ["Build Environments" page](https://portmaster.games/build-environments.html) describes several methods for cross-compilation through emulation of the target architecture.
[^2]: See [Debian Cross Toolchain overview](https://wiki.debian.org/ToolChain/Cross)
[^3]: "aarch64-linux-gnu" is what's known as a "triple/triplet" in cross-compilation, describing tersely the target environment. Read more about triplets [here](https://learn.microsoft.com/en-us/vcpkg/concepts/triplets)
[^4]: [Debian multiarch guide](https://wiki.debian.org/Multiarch/HOWTO)
[^5]: [Cross compiling with CMake](https://cmake.org/cmake/help/book/mastering-cmake/chapter/Cross%20Compiling%20With%20CMake.html)
[^6]: [Cross-compilation using clang](https://clang.llvm.org/docs/CrossCompilation.html)