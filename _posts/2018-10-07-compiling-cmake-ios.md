---
layout: post
title: "Compiling C++ + SDL2 + Vulkan application for iOS using CMake"
categories: development
comments: true
---

# Compiling C++ + SDL2 + Vulkan application for iOS using CMake

Recently I got the dumb idea to start deploying my work-in-progress game engine for iOS devices. I had in my naivety assumed that this would be an easy process, I was wrong. This post details how I managed to build applications written in C++, using SDL2 and Vulkan (MoltenVK), with CMake for iOS.

## Developer account necessary

First of you are going to need a paid Apple developer account (99$ a year). Apple allows self-signing applications using a "personal" account, I could not however get this working with the CMake toolchains. I assume it is possible, but after many hours of attempts I kinda gave up on that idea. With a paid account it works, at least for me.

Once you have your developer account, the first thing you need to find is your `Team ID`. At the time of writing it can be found under `Membership` on [developer.apple.com/account/](https://developer.apple.com/account/). The `Team ID` is a short string made up of seemingly random characters and numbers, e.g.: `2R23RTVQ3Y` (made up example).

Next you are going to need to create an `App ID`, this can be done under `Certificates, Identifiers & Profiles`. There seem to be a couple of different options here, but what has so far worked for me is to create one with my domain and a wildcard in it. I.e. `com.skipifzero.*`

![](/assets/posts/2018-10-07-compiling-cmake-ios/app_id.png)

## Polly toolchain

CMake does not yet have native iOS support, so we need a toolchain file. It seemed like the [Polly](https://github.com/ruslo/polly) toolchain is among the most well-supported ones, so that is what I choose to use.

In order to use Polly we have to install it to our computer and set a few environment variables. I choose to clone down the polly git repo to a permanent location (my dev folder) on my computer. This way I can update polly using `git pull` if necessary.

Now we have to set a few environment variables in our `.bash_profile`. I personally added the following ones:

```sh
export POLLY_ROOT="/Users/petorsfz/dev/polly"
export POLLY_IOS_DEVELOPMENT_TEAM="2R23RTVQ3Y"
export POLLY_IOS_BUNDLE_IDENTIFIER="com.yourdomain.*"
```

`POLLY_ROOT` is simply the path to where I cloned Polly on my computer.

`POLLY_IOS_DEVELOPMENT_TEAM` is your `Team ID` of your Apple developer account.

`POLLY_IOS_BUNDLE_IDENTIFIER` is the `App ID` you created.

## Build something!

At this point it's a good idea to try out if we can build and deploy a simple `C++` hello world program.

As a first step, if you haven't already done so, is to start `Xcode` and create a simple single view ios app. If you can't manage to deploy it to your device then something is messed up with your app signing and/or general setup, you should not continue before you have solved this.

If all is good so far we can try to deploy a simple project, e.g.:

`Main.cpp`:

```cpp
#include <cstdio>
int main(int argc, char* argv[])
{
	printf("Hello World!\n");
}
```

`CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.12 FATAL_ERROR)
project(helloworld LANGUAGES CXX)

add_executable(helloworld Main.cpp)
```

Then create the Xcode project using:

```sh
mkdir build_ios
cd build_ios
cmake .. -GXcode -DCMAKE_TOOLCHAIN_FILE=$POLLY_ROOT/ios.cmake
```

Hopefully everything should build and sign correctly at this point, at least it did for me.

## SDL2

Annoyingly `SDL2` does not ship pre-built binaries for iOS, so we have to build them ourselves.

First download the `SDL2` source from the [official download page](https://www.libsdl.org/download-2.0.php). After unpacking there should be a directory called `build-scripts`, inside this directory there is shell script called `iosbuild.sh`. Run this script to build, the output should be inside the `lib` folder in the same directory.

Now you have to choose how you want to handle the SDL2 dependency in your project. Personally I just uploaded the built libraries to my own [CMake friendly SDL2 distribution](https://github.com/PhantasyEngine/Dependency-SDL2), which I then include in projects that use SDL. I'm going to leave the details out of this blog post as I don't aim to teach how linking works in CMake, but you can check out my distribution above for some hints.

At this point we are done and can start using SDL2 in our application right!? Unfortunately not, I had some weird linker issues at this point. The solution for me was to add few extra things to the linker when building for iOS, i.e:

```cmake
if (IOS)
	target_link_libraries(SDL2-iOS-Test "-framework AudioToolbox")
	target_link_libraries(SDL2-iOS-Test "-framework AVFoundation")
	target_link_libraries(SDL2-iOS-Test "-framework CoreGraphics")
	target_link_libraries(SDL2-iOS-Test "-framework CoreMotion")
	target_link_libraries(SDL2-iOS-Test "-framework Foundation")
	target_link_libraries(SDL2-iOS-Test "-framework GameController")
	target_link_libraries(SDL2-iOS-Test "-framework OpenGLES")
	target_link_libraries(SDL2-iOS-Test "-framework QuartzCore")
	target_link_libraries(SDL2-iOS-Test "-framework UIKit")
	target_link_libraries(SDL2-iOS-Test "-framework Metal")
endif()
```

I'm sure this makes sense if you are an experienced iOS developer. But for me this left me scratching my head a bit. Whatever, it works.

## Vulkan

Apple has choosen to not support Vulkan officially, instead focusing on Metal. Luckily, there is a Vulkan implementation on top of Metal called [MoltenVK](https://github.com/KhronosGroup/MoltenVK). Therefore, in order to use Vulkan on iOS we need to link with MoltenVK.

A VulkanSDK for macOS/iOS utilizing MoltenVK is available via [LunarG](https://vulkan.lunarg.com/). I choose to install this via brew cask:

`brew cask install vulkan-sdk`

In order to hook it up properly it is also necessary to set a couple of environment variables, I used the following ones:

```sh
export VULKAN_SDK=/usr/local/Caskroom/vulkan-sdk/1.1.82.1/macOS
export VK_ICD_FILENAMES=$VULKAN_SDK/etc/vulkan/icd.d/MoltenVK_icd.json
export DYLD_LIBRARY_PATH=$DYLD_LIBRARY_PATH:$VULKAN_SDK/lib
export VK_LAYER_PATH=$VULKAN_SDK/etc/vulkan/explicit_layer.d

# Not necessary, but makes glslc (shader compiler) available on command line
export PATH=$VULKAN_SDK/bin:"$PATH"
```

With this the native CMake `find_package(Vulkan)` works perfectly when building for macOS, it's not good enough for an iOS build though. In order to link on iOS I do the following:

```cmake
set(MoltenVK_Path $ENV{VULKAN_SDK}/..)
set(Vulkan_INCLUDE_DIRS ${MoltenVK_Path}/MoltenVK/include)
find_library(MoltenVK_LIB
    NAMES
    MoltenVK
    HINTS
    ${MoltenVK_Path}/MoltenVK/iOS)
```

## File access

One of the first things that tripped me up once I got everything compiling and running correctly was files. You are not allowed to read and write files however you want in an iOS application. It was also not obvious to me how to bundle files with the application onto the iOS device itself.

The solution I finally came up with after a lot of searching and experimenting is the following:

```cmake
add_custom_command(TARGET helloworld POST_BUILD
	COMMAND ${CMAKE_COMMAND} -E copy_directory
	${CMAKE_CURRENT_SOURCE_DIR}/res
	$<TARGET_FILE_DIR:helloworld>/res
)
```

The above copies the directory `res` in the root of my project (next to the `CMakeLists.txt`) into the iOS application bundle. The path to application bundle where this res directory resides can easily be queried using `SDL_GetBasePath()` if necessary.

## Conclusion

Unless you are developing a cross-platform project targetting many platforms, don't bother. I haven't tried it, but I get the feeling going for a pure Xcode based setup would be way easier. With that said, I think this turned out quite well.

We have somehow accomplished to get SDL2 + Vulkan + C++ running on iOS using a CMake based build system. I am convinced some of the solutions I have choosen are stupid. However, with the lack of good documentation/tutorials for how these kind of things should be done, I think this is a fairly good start.

To be completely honest, I'm somewhat baffled by this whole affair. Getting everything described in this post working has taken more than 15 hours of work time. I don't feel like anything I have done here should be particularly unusual, but it has been very hard to find good information on how to accomplish it. I'm hoping this post can be of use to someone who are stuck in a similar position.
