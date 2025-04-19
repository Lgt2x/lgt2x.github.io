+++
date = '2025-04-16T23:33:13+01:00'
draft = false
title = 'One Year of Descent 3 Open Source'
+++

One year ago, the engine source code of the 1999 6-axis shooter Descent 3 was released as Open Source software, under the GPLv3 license. This post recounts the community developments that followed this release, where the game stands today, and where it is headed.

## Early times

When the news broke and spread across the web, a large attention was drawn to the project, who found headquarters on the Descent Developpers Discord. Kevin Bentley, an original developper of Descent 3, and the one to release the game, spent time to solidify the community around a common fork and recruited a small team of volunteers to help merge Pull Requests (PR) coming in. 

Quickly, people got the game building on Windows, then Mac and Linux. A CI pipeline was set up to keep the game in a clean building state across PRs. The first days were pure chaos. In a week, about a hundred PRs were submitted, mostly build, CI, documentation and code style fixes. It was relentless for maintainers of the project (which I was part of)

## Towards a first release



## Piccu Engine

While a large part of the community was busy slowly discovering the newly released code, veteran Descent hacker InsanityBringer went on to build their own fork which would

While most of the community

## Ports, ports ports

Opening the game's source also led to a variety of ports by different community members. The first notable one being the PortMaster version of Descent 3, made to run on Linux handheld devices. At the time, the engine still used legacy OpenGL 1 calls for graphics, which is generally not compatible with modern handheld devices that have OpenGLES drivers. PortMaster's general solution to this issue is using the [GL4ES compatibility layer](https://github.com/ptitSeb/gl4es), translating GL1 calls to GL ES. This works pretty well, at the cost of a slight performance hit.

![]()
Image credits: JeodC

In the same vein, an experimental WebAssembly port was created, also leveraging GL4ES, because Emscripten does not emulate OpenGL 1.5 correctly either. This port was not very practical, because required game assets are about 600MB, which is a lot to be transferred from a web server. This port could be revisited for better performance now that the engine uses OpenGL2 and SDL3! Some work could also be done on reducing or compressing game assets, reducing them to their bare minimum, so that a web server can transfer them in a reasonnable time (bar the legal issues with downloading game assets).

![]()

Later, an Android port was created, using yet another method of managing game assets: the user needs to upload them to a web server running on the Android device, so they are copied to persistent storage. Controls on a phone/tablet are also an issue that has not been solved yet: a 6 axis shooter requires quite a lot of different buttons to be played, and the game basically assumes that players have access to a full keyboard, also using function keys. A phone has, well, a touchscreen, and not a lot of function keys. A full native port would need the whole interface to be thought through again, and on-screen buttons to be created, all of that without clogging too much the view. The Android port was also made possible by the implementation of ARM64 cross-compilation pipelines originally created for the PortMaster build.

There is still a lot to do when it comes to porting and hardware compatibility, but all of that is hugely facilitated by the SDL3 back-end, which offers compatibility with basically any platform one can think of. As we made the choice early on to drop 32-bit compatibility, [...]

## Challenges

**Third-party management**: C++ is one of the most widely used languages today without a standard 3rd-party dependency management system. Depending on the project, you may find git submodules, CMake ExternalProject, raw binaries in the repository (??), or any of the nascent solutions at a unified package manager, such as Conan or VCPKG. And did I mention that every developper has a *strong opinion* about it and thinks their solution is the best? At the start, Descent 3's only 3rd-party libraries were zlib (embedded in the code), mvelib for video playback, and SDL for Linux build. Over time, we figured out that a logging library would be handy, as well as a HTTP library for downloading level maps online. After some wandering, we chose VCPKG as the main dependency management system, because of its cross-platform support, easy integration into CMake and wide range of available libraries. This was not a tool I was personnally familiar with at the start, but eventually built up familiarity. While Windows and MSVC integrations are near flawless, I find Linux support to be a little more rough around the edges; system package requirements to run the libraries build are sometimes obscure and undocumented. However, the triplet system has proven very handy to manage cross-compilation to ARM64, and to produce MacOS Universal (x86+x64) builds. VCPKG loads as a toolchain in CMake and provides the targets for the required libraries. This also means that it is trivial for any Linux packager to swap out VCPKG for system packages, given the versions are compatible: either use VCPKG to provide libraries, or let CMake's FindPackage look on the system for the library.

**Assets packaging**: As only the source code for Descent 3

**Managing expectations**

## Non challenges

Some things could have been challenging if done differently, but luckily were not.

**Cross-platform support**: the version of the engine code we were provided used CMake as the build generation system. This allowed the community to quickly get builds going on all platforms.
The original code contained quite a lot of platform-specific code, using SDL1.2 for Linux, and DirectX for Windows. Icculus, who originally ported Descent 3 to Linux in 2000, generously contributed patches from his [2020 SDL2 port of the game](https://www.patreon.com/posts/project-descent-33611585) to the GPL upstream. Getting a SDL2 port meant that the Windows-specific DirectX code could be directly be removed to get unified code for all I/O and graphics. SDL really is a god-send for easy cross-platform support, and abstracted away all low-level OS primitives so we barely have any platform-specific code left yet. The game settings, which used to be stored in the Windows registry, are now stored in a portable configuration file on all OSes.

**Community buildup**: 

**Legacy code**: 

## Where we stand now


## Looking forward

While a lot of work has been done so far on Descent 3, there are still a lot left to do.

**Improved controller support**

**Level scripting**

**Level editor**

**Unified Descent launcher & settings**

**Competitive Descent**

**Video playback**

**Distribution packaging**

If you wish to
## Personal impressions

For me, this was a first experience at Open Source project management at this scale. This is overall an exciting yet intimidating experience. Exciting because of all the activity and cool ideas around the project, but intimidating because of all the developpers way more experienced than me around it.

Confidence built up as my knowledge of the codebase increased. It was harder at the start to be confident in reviews for code I had not seen before, but discussing contributions with other maintainers and trying to apply general guidelines worked well enough until I was familiar with the global architecture and the direction we wanted to go in.

My involvement over this past year has been consistently inconsistent; as work and life went, I could not be responsive and active on the project all year round, sometimes having to stay in the darkness for a month at a time for various reasons. Descent 3 has been my only hobby coding project this past year, and I'm not ready to move on yet, given all the work there is left!

