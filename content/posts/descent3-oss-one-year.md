+++
date = '2025-04-16T23:33:13+01:00'
draft = false
title = 'One Year of Descent 3 Open Source'
+++

One year ago, the engine source code of the 1999 6-axis shooter Descent 3 was released as Open Source software, under the GPLv3 license. This post tells the community developments that followed this release, where the game stands today, and where it is headed.

## Early times

When the news broke and spread across the web, a large attention was drawn to the project, who found headquarters on the Descent Developpers Discord. Kevin Bentley, an original developper of Descent 3, and the one to release the game, spent time to solidify the community around a common fork and recruited a small team of volunteers to help merge Pull Requests (PR) coming in. 

Quickly, people got the game building on Windows, then Mac and Linux. A CI pipeline was set up to keep the game in a clean building state across PRs. The first days were pure chaos; in a week, about a hundred PRs were submitted, mostly build, CI, documentation and code style fixes. It was relentless for maintainers of the project, who had to figure out which contributions to accept, as well as the general direction to give to the project. Of course, some mistakes were made, the first one being to format the entire codebase using clang-format very early on. While this seems like a great idea at first, it had the unintended consequence of creating major conflicts with people's forks, which were based on the non-formatted code. In particular, this slowed down porting Icculus' patches, whose 2020 steam port has many improvements not included in the released code. There was also some back and forth on dependency management, caused by a lack of a clear management and agreement between maintainers.
![]()

## Towards a first release

After a rough first few months, when everyone in the community had a stable build and could play the game on their favorite OS, it was time to get a first release going. For this first released open source version of Descent 3, we set clear goals: full 64-bit compatibility, playable on Windows, Linux and MacOS. This release would not have any gameplay or graphics improvements over the commercial game, but would be a stable base for hackers to play with. Most dead code supporting outdated hardware was removed, and platform-specific code was reduced to a minimum. This 1.5 release was tagged in August, about 4 months after the code release. It was well received by the community, although some players had difficulties setting up the game. This is something we would focus on for the next version.

## Piccu Engine

While a large part of the community was busy slowly discovering the newly released code, veteran Descent hacker InsanityBringer went on to build their own fork which would become known as "Piccu engine". InsanityBringer was already known as the author of the "InjectD3" mod, which brought a lot of quality of life features to the aging D3 game, and became the de-facto standard for most players. This effort was 

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

**Assets packaging**: As only the source code for Descent 3 was released under an open source license, assets are still proprietary. Players need to either get the game from an online shop (Steam, GoG), or install it from a CD and patch it to the 1.4 patch in order to get game assets. Not packaging the assets means that open source ports need to implement a different logic to find them. Our first 1.5 release assumes that assets can be found in the same directory as the game executable, just like the commercial game does. This creates extra steps for the player, who needs to find assets location from his commercial D3 installation, and copy them over to the port's directory. This operation may be unfriendly to some users, who'd hope for an easier setup. So, why not look for the base game installation to automatically find assets? Steam lets the user select individually each game's installation path, and has no standard API to retrieve it. The same problem occurs for a CD installation as well. This is not a solved problem right now, but the current solution is looking in classic Steam/GoG installation paths for assets, as let the user select the actual installation path from a GUI selection window otherwise.

**Managing expectations**

## Non challenges

Some things could have been challenging if done differently, but luckily were not.

**Cross-platform support**: the version of the engine code we were provided used CMake as the build generation system. This allowed the community to quickly get builds going on all platforms.
The original code contained quite a lot of platform-specific code, using SDL1.2 for Linux, and DirectX for Windows. Icculus, who originally ported Descent 3 to Linux in 2000, generously contributed patches from his [2020 SDL2 port of the game](https://www.patreon.com/posts/project-descent-33611585) to the GPL upstream. Getting a SDL2 port meant that the Windows-specific DirectX code could be directly be removed to get unified code for all I/O and graphics. SDL really is a god-send for easy cross-platform support, and abstracted away all low-level OS primitives so we barely have any platform-specific code left yet. The game settings, which used to be stored in the Windows registry, are now stored in a portable configuration file on all OSes.

**Community buildup**: unlike most open source projects starting from scratch, Descent 3 benefitted from the support of an existing community, already experienced with the game and the series as a whole. New people kept coming in and contributing to the effort without any additional external communication effort. It's always great to have people willing to help test patches at any moment!

**Legacy code**: The Descent 3 code we inherited was barely C++98, more like C with classes, using exclusively the C standard library. While it does not compare with modern C++20 practices, compatibility with current day compilers is still great and did not cause too much trouble. One cannot dismiss how well 25 years old C++ code holds up to this day! Overall, this legacy code is not a huge hassle to maintain. Some sections have been reworked to use STL containers, standard strings and smart pointers, but this can be done gradually, without breaking the whole structure.

## Where we stand now



## Looking forward

While a lot of work has been done so far on Descent 3, there are still a lot left to do. We list here the main areas where Descent 3 could be improved.

**Improved controller support**: 

**Level scripting**: probably the biggest challenge out there. Descent 3 levels can execute scripts, that are binaries embedded in the game assets. These scripts can add logic around doors, triggers, etc. Scripts for levels from the main campaign have been released alongside the game engine code, so we could compile them and make sure they can be run. We build a custom HOG [^1] file with a dynamic library for each level of the campaign: a `.dll` on Windows, `.so` on Linux/BSD and `.dylib` on MacOS. This works well, except for user-made levels, that can also be scripted using the level editor. In this case, a Windows 32-bit DLL is generated and embedded in the level HOG file. As a consequence, only Windows 32-bit clients will be able to run the level script properly. The Descent 3 community port does not build 32 binaries anymore, which means that user-created scripts cannot be run anymore. As a side-note, this is the main reason that the Piccu engine port is still 32-bit only to this day. Being able to properly run user-created levels is critical from a game preservation standpoint, and it is currently the one major limitation that the 64-bit community port has. There is also a security concern associated to it: online Descent 3 server can send levels to clients connecting to it, including the script DLL, that is therefore executed on the client. Yes, it sounded fine to run random binaries on people's computers in '99, but today, that would be considered absolutely critical today. The Descent 3 community port has totally disabled the execution of scripts from levels outside of the main campaign to prevent any exploit.

Most of the user-created stages over the years were distributed without the script source code available, so they likely cannot be compiled anymore. We are left with a Win32 x86 binary blob that contains compiled C++ code, that we would need to somehow sandbox and run securely on any OS. Some suggestions have been made to make this possible, but nothing concrete has been done yet.

**Level editor**: The source code for D3Edit, the Descent 3 level editor was also made public; to my knowledge, nothing much has been made to improve the Descent 3 level edition experience starting from D3Edit, which is an old MVC app. It may be more valuable to use the amazing Inferno Descent Editor and add the necessary primitives to make it compatible with Descent 3. 

**Unified Descent launcher & settings**: Another idea that has been discussed before, is that now that all first 3 Descent games have open source ports, it would be nice to have a unified UI launcher that can look for game files and try to copy control configurations between games. This would also benefit to installers such as PortMaster. 

**Competitive Descent**: There is a small but dedicated playerbase for competitive Descent 3, that has long been waiting for improvements to the "less casual" part of the game. There are a lot of improvements to be done on the networking code, server deployment and configuration and controls to make it a better experience overall. Why not a physical LAN someday?

**Linux Distribution packaging**: After the first release of Descent 3 came out, Linux users started packaging the game for their distribution. To my knowledge, there is currently an OpenSUSE and NixOS package, and the game is listed on FreeBSD games. If you're a packager and wish to ship the game for your distribution, feel free to do so!

**Free Asset pack**: Unlike the previous Descent games, Descent 3 does not have a shareware distributable version, so players cannot run the game at all without a the complete set of proprietary assets. While this would be a high-effort task, for easier distribution and testing of the engine, people suggested creating new compatible assets from scratch, under a permissive license.

If you wish to help solving one of these, please reach out to us!

## Personal impressions

For me, this was a first experience at Open Source project management at this scale. This is overall an exciting yet intimidating experience. Exciting because of all the activity and cool ideas around the project, but intimidating because of all the developpers way more experienced than me around it.

Confidence built up as my knowledge of the codebase increased. It was harder at the start to be confident in reviews for code I had not seen before, but discussing contributions with other maintainers and trying to apply general guidelines worked well enough until I was familiar with the global architecture and the direction we wanted to go in.

My involvement over this past year has been consistently inconsistent; as work and life went, I could not be responsive and active on the project all year round, sometimes having to stay in the darkness for a month at a time for various reasons. Descent 3 has been my only hobby coding project this past year, and I'm not ready to move on yet, given all the work there is left!

