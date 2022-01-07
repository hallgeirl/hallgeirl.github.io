---
title:  "Digging up my old C++ game project"
date: 2020-09-20 11:08
categories: cpp gamedev
redirect_from:
  - /2020/09/20/i-dug-up.html
---


I dug up some  of my old coding projects this weekend. Oh boy, there's lots of weird stufff I've been making over the years, especially in my early youth. But there's some cool things as well. For instance, I made a Mario clone in C++ complete with a map editor and everything. Trying to get this project to compile were not simple 12 years later. Libraries were missing, were hard to find, and had moved on to newer versions. My environment has changed - back then I mostly used Linux, but for various reasons I now use Windows. With that I'd like to reflect on a couple of topics that I came to think about during the process of getting my project up and running again.

## The newer C++ standards are quite awesome
Back in 2008, C++11, and certainly not C++17, were obviously not written yet. I had to rely on Boost libraries for things like smart pointers. There were no "foreach"-like syntax. Now, iterating arrays are just as easy as in other languages that include a foreach-syntax:

```cpp
vector<string> foo;
foo.push_back("hi");
foo.push_back("there");
for (const auto& val : foo)
{
  cout << val << endl;
}
```
This seems super trivial today, where this kind of syntax is standard, and I would argue it's more or less expected from a language today to include this kind of syntax. Back in 2008 however, having most of my experienced in C++, this was not the case. Smart pointers are also a very welcome addition to the C++ standard family!

Additionally, much functionality has been added to the standard libraries. To mention a couple that I've started using now, are std::filesystem and std::chrono for directory traversal and frame timing in my game, respectively. I had to rely on boost for a lot of things, and timing functions were mostly platform specific.

## Garbage collection? I don't miss you
In .NET, Java, Javascript and many other languages, you have a garbage collector running for you to ensure that objects you allocate on the heap are deallocated when no one are referencing them anymore. This is great - but it does have a performance cost. For large projects with a lot of object allocations, the garbage collection cycle can negatively impact performance. 

With C++, you have no garbage collector. You need to manage the heap yourself. If you have a "new", you need a "delete", or else you get a memory leak. However, a common best practice in C++ is to avoid dynamic memory allocations as much as you can, which significantly limits this problem, and in some cases even removes it completely. Instead, C++ objects are allocated on the stack. Objects on the stack gets deleted when the stack frame is "done". For instance, local objects defined in a function are deallocated when the function returns (unless you allocate objects on the heap, which you shouldn't). 

So what's great about this? Well, the obvious one is that you no longer NEED a garbage collector. I find it strangely comforting to know exactly how long my objects live - and I don't need to hypothesize if the high memory usage of my application is because the garbage collector hasn't run yet, or if I actually have a memory leak. Stack allocation just is so comfortable. It's not perfect if you need heap-allocated objects, however. I will aim to get rid of most or all heap allocations though.

## CMake
I based my old game project on Makefiles. This is great - if you're running on Linux. But Makefiles haven't really been the "standard" on Windows. I'm sure you could use them somehow, though. But I wanted to find a proper cross-platform alternative - and that alternative is CMake. CMake is great - it's fully supported in Visual Studio, is cross-platform and, in my opinion, has a much cleaner syntax than Makefiles. Here's a minimal example:
```cmake
cmake_minimum_required (VERSION 3.8)

add_executable (CMakeTest "CMakeTest.cpp")
```
This doens't do much of course, other than compiling CMakeTest.cpp into an executable CMakeTest. It's super simple to also add include paths and link paths to the target. 

CMake were around in 2008, but I was not aware of it, or perhaps didn't see the value. I'll be using CMake from now on. 

## Cross-platform is king
People sometimes switch operating systems, for various reasons. I switched from Linux to Windows after I started working in my current organization, which is mainly a Windows shop. I also enjoy gaming, and back then gaming on Linux were not that mature (and it still has its issues). Anyway, I made my game project to be cross platform from the start using SDL. This was one of the best decisions I made for this projects I think - because it allowed me to get it up and running on my current machine without TOO much hassle.

In general, I really want software to be cross-platform. Why shouldn't it be? WHY are for instance game developers NOT making their games available on Linux, when it's really not that hard? C++ is quite portable, after all. Anyhow, for me, this makes a lot of sense. I believe that if you work towards cross-platform compatibility you will also have a better, more standardized code base, because you can't rely on platform-specific hacks to get it to work.

We're working hard to make our product cross-platform in my professional work as well.

## Dependency management in C++ is still a nightmare :-)
I've been spoiled the last few years. I've worked with .NET, and here we have a solid package manager called Nuget. Do you need a new dependency, for instance a JSON library? Install it with Nuget, and it'll work for everyone using this project. 

C++ is a different beast. Perhaps with C++20 and the new concept of "modules", there's a chance of some innovation on this area, but currently it's NOT possible to just "download, build and run" a project. You need to download the project, then install all sorts of dependencies, and their dependencies. For instance, "luabind" which I use in my project, depends on boost. boost itself is HUGE, but it needs to be installed. I need to install SDL. Then I need to install SDL_Mixer and the LUA libraries. Then MAYBE it'll work. And this is my own SMALL project.

C++ needs a standard way of handling dependencies, in my opinion. I have a strong opinion that ALL you should need to do to build a project, is to download it from github, and run CMake (or whatever build tool you got). Dependecies should be fetched automatically.

The way to achieve this today would be to either include the source in your project (directly, or as git submodules), or to include libraries for each platform in the repository itself. The first option is probably the most portable - but perhaps not all libraries HAVE the source available. And there could be a LOT of dependencies, because dependencies has dependencies.

What many does however is to require that dependencies are installed on your system before compiling. For Linux users, this is probably not a bad way to go, as you can typically get a hold of these dependencies using your package manager. For Windows however, this can be a daunting task, to hunt down binaries and headers for all libraries that you need, and configure include and linker paths for these.

In my own project, this is an unsolved problem however. Some libraries (LUA and luabind) are included. Some are not.

## How little I knew...
Since 2008, I've worked 8 years as a professional software developer. Before this, my programming experience were my own hobby projects starting in elementary school and onwards, as well as academic projects. There's quite a leap going from these smaller projects, to giant enterprise-level software that NEEDS to be up and running for hundreds of thousands of users, every single hour of every single day. It's fun to go back to look at my older projects and see how my coding style has changed over the years. 

## Wrapping up
Well... I don't expect anyone to learn a lot of new things here, since I'm discussing my own ancient project and my reflections around this, but I'm always eager to discuss any of these topics. Feel free to reach out on Twitter!

For the record - here's my coveted game project. It builds and runs (at least on Windows at the moment) after I switched to CMAKE and included some dependencies. You will have to install boost as well though. For Linux, you will have to install SDL and SDL_Mixer too.

[https://github.com/hallgeirl/hiage-original/tree/master](https://github.com/hallgeirl/hiage-original/tree/master)

I now look forward to taking up my old game project again to modernize it and extend it!