---
layout: post
title: "Single-chunk game state - Part 2 - The implementation details"
categories: development
comments: true
---

# Single-chunk game state - Part 2 - The implementation details

This is the follow-up post to [my previous one](http://www.skipifzero.com/development/2018/10/07/compiling-cmake-ios.html), in which I discussed [Phantasy Engine](https://github.com/PetorSFZ/PhantasyEngine)'s game state in very general terms.

To be frank, I'm not completely happy with how that post turned out. I had really planned to fill it with the content that is now in this post. But it got too long, so I felt like I had to cut it down somehow. Which meant basically cutting out the technical details, which was probably the most interesting part of the whole thing. Oops.

Anyway, having read the previous post will probably make it easier to understand this one. This will still be a very technical post, at least basic knowledge of `C` and `C++` is recommended. Let's dig in!

## Some context

Before we start I'm quickly going to cover some related technical topics. Some are just why we use or don't use certain `C++` language features, other are more advanced topics which you might not normally come across even if you regularly program in `C++`.

### `C++` templates (and how we use them)

[`C++` templates](https://en.cppreference.com/w/cpp/language/templates) is `C++`'s metaprogramming feature. It essentially allows the programmer to write generic code that can then be compiled into specific code for specific parameters. `C++` templates are generally terrible in many aspects. A short list:

1. They are more complex and harder to read than normal code. [KISS](https://en.wikipedia.org/wiki/KISS_principle).

2. They take longer to compile than normal code, sometimes many times longer.

3. Because of how `C++` compilation works, the template [definition](https://en.cppreference.com/w/cpp/language/definition) usually need to be in a header file (`.h`) instead of a compilation unit (`.cpp`). This means that the template need to be compiled for each compilation unit (`.cpp`) it is used in, multiplying the cost of compilation.

4. They are hard to debug. The errors generated can be ridiculously hard to read and understand.

5. They have a tendency to spread and infect the rest of your codebase. If you have a templated type which you want to operate on with a function, you usually have to make that function a template itself so you can operate on all variants of that type. Etc.

The last point (5) is especially nefarious. It basically means that once you start using templates you usually have to use even more templates, and it sort of accelerates from there. Combined with points 2 and 3 you easily end up with a codebase that takes way too long to compile, not fun.

Templates are not all bad though. Used in a responsible way they can greatly improve your code. A personal rule I have discovered through trial and error is to avoid template types, try to only use template functions and methods.

In other words, don't do this:

```cpp
template<typename T>
class TemplatedClass {
  // ...
};
```

But feel free to do this (if appropriate and necessary):

```cpp
template<typename T>
void templatedFunction(T a, T b)
{
  // ...
}

struct NonTemplatedStruct {
  template<typename T>
  void templatedMethod(T a, T b)
  {
    // ...    
  }
};
```

The reason for this is that, in my experience, templated types is what mostly causes the templates to spread all over your codebase. Templated functions and methods don't really do that the same way. It is also slightly more limited what you can (easily) do with templated functions, so they tend to stay (at least a bit) simpler.

### Memory alignment

Unless you have done any low-level programming, such as [SIMD](https://en.wikipedia.org/wiki/SIMD) programming, this is probably not something you have any experience with. Memory alignment means what memory (pointer) addresses a type is allowed to be located at.

As a simple example, on some CPU architectures a 32-bit integer (such as `uint32_t`) can only be read from 4-byte aligned addresses. I.e., the address need to be a multiple of 4. An address of `404` ($$ \frac{404}{4} = 101 $$) would be fine, and address of  `405` ($$ \frac{405}{4} = {101.25}$$) would not be fine.

Normally no types have a higher alignment requirement than 4-bytes. However, SIMD types usually require 16-byte or even 32-byte alignment for optimal use. The alignment of a specific type can be checked with the [`alignof()`](https://en.cppreference.com/w/c/language/_Alignof) operator, which works similarly to [`sizeof()`](https://en.cppreference.com/w/cpp/language/sizeof).

The main takeaway for now is that you can't just place arbitrary structs at arbitrary locations in memory, they have to be aligned to memory addresses that fulfil the type's requirements.

### POD types

A [POD (Plain Old Data)](https://en.wikipedia.org/wiki/Passive_data_structure) type is a type which:

* Has default constructor, copy constructors and destructors.
  * I.e., same as no constructor and destructor.
  * Can be trivially copied using [memcpy()](https://en.cppreference.com/w/cpp/string/byte/memcpy) and similar.
* Everything is public, no private members. Usually a `struct` in C++.
* All members are also POD.

POD types are very useful. You can essentially think of them as a pure way of storing data, without adding any logic or constraints to the data. The most useful properties for our purposes is the fact that they don't need any cleanup when destroyed (i.e. no destructor) and that they can be trivially copied without any custom logic.

You can check if a type is POD by using [`std::is_pod<T>`](https://en.cppreference.com/w/cpp/types/is_pod):

```cpp
#include <type_traits>
static_assert(std::is_pod<int>::value, "Compile error if int is not POD");
```

However, POD is actually too strong of a requirement for us. Namely because it interferes with a feature that was added in `C++11`, the ability to default initialize members in a `struct`:

```cpp
struct AlmostPodStruct {
  int a = 3;
  int b = 2;
};

AlmostPodStruct s;
assert(s.a == 3);
assert(s.b == 2);
```

The above feature is immensely useful. If nothing else, it easily documents the expected default values for each member. The downside is that `AlmostPodStruct` is no longer a POD, because it now has a non-default constructor.

The thing is, `AlmostPodStruct` is almost a POD. In fact, it has all the previously mentioned properties we care about. So we are going to do something ugly. We are going to pretend that it is POD. Instead of checking if a type `T` is POD, we are going to check the following:

```cpp
#include <type_traits>
static_assert(std::is_trivially_copyable<T>::value, "Check if memcpy works");
static_assert(std::is_trivially_destructible<T>::value, "Check if no destructor");
```

This, of course, means that we skip initializing the struct's members with the specified default values inside the game state. That sucks a bit, but it is fine honestly. This way we can at least rely on the default values when creating structs outside the game state.

### Relocability

Relocability is the ability to copy stuff around in memory and have them work, regardless of which memory address they are on. POD types mostly fulfill this already, with one caveat, pointers. Below is a small example where it breaks;

```cpp
struct StructWithPointer {
  int value;
  int* ptrToValue; // Pointer to value above
};

StructWithPointer a;
a.value = 2; // Initialize value to 2
a.ptrToValue = &a.value; // Make pointer point to value

StructWithPointer b = a; // Copy first struct
b.value = 3; // Change value

int output = *b.ptrToValue; // Get value pointed to by b's pointer
// The value of output is 2, because b's pointer points to a's value.
```

The above example should be quite clear. Pointers get invalidated when stuff move around in memory. And since we want to move stuff around in memory we simply have to avoid them.

What can you do instead? Use offsets. An offset, often expressed in bytes, is a relative distance from some known pointer. The `this` pointer is very useful for this purpose.

### Strict aliasing

In `C++` it is actually [undefined behavior to access memory of one type as another type by reinterpret casting a pointer](https://en.cppreference.com/w/cpp/language/reinterpret_cast#Type_aliasing). This is of course stupid and makes it very hard to do any cool stuff. The Visual Studio compiler does not utilize strict-aliasing for anything, but if you are building with GCC or Clang you should pass the `-fno-strict-aliasing` flag to disable these optimizations.

If you intend to use techniques I describe in this post it is not really a suggestion, it is a requirement. A lot of the stuff I do is not allowed under the strict-aliasing rules.

## ArrayHeader

We are finally done with the background, let us start talking about the game state itself. As mentioned in the previous post, there are quite a lot of arrays stored in the game state. One array for each component type, one array for the generations, one for the component masks, etc. This section describes `ArrayHeader`, which is used to keep track of the arrays in the game state.

### Memory layout

In standard C++ there are two common ways of storing arrays, a plain `C`-array (alternatively [`std::array<T,N>`](https://en.cppreference.com/w/cpp/container/array)) or [`std::vector<T>`](https://en.cppreference.com/w/cpp/container/vector) (alternatively its [cooler replacement](https://github.com/PetorSFZ/sfzCore/blob/master/lib-core/include/sfz/containers/DynArray.hpp) in [sfzCore](https://github.com/PetorSFZ/sfzCore)). The memory layout of a `C`-array is as follows:

![](/assets/posts/2019-05-02-single-chunk-game-state-part2/array_memory_layout.png)

The array is placed directly in contiguous memory on the stack. You then pass around pointers to this piece of memory when using the array in different places. A `std::vector<T>` on the other hand have the following memory layout:

![](/assets/posts/2019-05-02-single-chunk-game-state-part2/dyn_array_memory_layout.png)

The above layout is obviously not what we want since we only want a single chunk of memory. `std::vector<T>` does have a number of advantages over a `C`-vector though. Specifically that it keeps track of the current size and capacity, which makes it possible to have the convenience methods [`push_back()`](https://en.cppreference.com/w/cpp/container/vector/push_back) (or `add()` as I call it) and [`pop_back()`](https://en.cppreference.com/w/cpp/container/vector/pop_back). Also range-checking if you are into that sort of thing I guess...

What we want is a third option which is (as far as I know) not available in the standard library. This is what I call `ArrayHeader`, and the memory layout looks like this:

![](/assets/posts/2019-05-02-single-chunk-game-state-part2/array_header_memory_layout.png)

Essentially it is a fixed size `C`-array, but it has a header on top which keeps track of the current size and capacity. The `ArrayHeader` struct also has methods for manipulating the array, so we get (most) of the convenience with `std::vector<T>` in a more useful format!

### ArrayHeader implementation

The ArrayHeader is not your ordinary `C++` class. It has a number of tricks and unusual behaviors which are not that common in code you usually see. A simplified (and templated) implementation could look as follows:

```cpp
template<typename T>
struct ArrayHeader {
  uint32_t size;
  uint32_t capacity;
  uint8_t ___padding___[24];
  
  T* arrayPtr()
  {
    uint32_t offsetInBytes = sizeof(ArrayHeader);
    uint8_t* rawArrayPtr = reinterpret_cast<uint8_t*>(this) + offsetInBytes;
    return reinterpret_cast<T*>(rawArrayPtr);
  }
  
  T* elementAt(uint32_t idx) {
    if (idx >= size) return nullptr;
    return arrayPtr() + idx;
  }
};

static_assert(sizeof(ArrayHeader<T>) == 32, "32-byte alignment");
```

So the first thing you see and might surprise you is the `uint8_t ___padding___[24];` member. Remember earlier when we talked about memory alignment? Essentially I require that the ArrayHeader struct is exactly 32-byte big, so if the ArrayHeader itself is on a 32-byte aligned address then the array itself will be too.

The `arrayPtr()` method returns a pointer to the array (placed after the ArrayHeader in memory). Notice how we are utilizing the fact that the this pointer will always point to the start of the current ArrayHeader we are operating on to achieve relocability. An easy to make mistake here is to write `reinterpret_cast<uint8_t*>(this + offsetInBytes);` instead of `reinterpret_cast<uint8_t*>(this) + offsetInBytes;`. This will give you completely wrong results, watch out.

Now, let's improve this starting point a bit. Remember how we talked about the fact that templated types are bad? We are not actually using the type `T` for any of the members in ArrayHeader. As an alternative we could add a member to keep track of the size of each element, i.e.:

```cpp
struct ArrayHeader {
  uint32_t size;
  uint32_t capacity;
  uint32_t elementSizeBytes;
  uint8_t ___padding___[20];
  
  template<typename T>
  T* arrayPtr()
  {
    assert(sizeof(T) == elementSizeBytes);
    /* otherwise same as before */
  }
};
```

Here we utilize the size of each component as a sort of ad-hoc type-safety. Which is neat!

Another problem with the ArrayHeader is that it really isn't a normal struct. All its methods operate on data that lies outside the struct, but which you don't have an explicit pointer to. The following is a simple error a user could do:

```cpp
ArrayHeader* arrayPtr = ... // (Memory chunk with header and array)
ArrayHeader header = *arrayPtr; // Copies header but not array
header.someOperation(); // Invalid memory access because no array
```

In fact, if you are not operating on a pointer (or reference) to an ArrayHeader it is almost always an error. Luckily, C++ allows us to do something about this:

```cpp
struct ArrayHeader {
  /* ... */
  // Disable the copy constructors
  ArrayHeader(const ArrayHeader&) = delete;
  ArrayHeader& operator=(const ArrayHeader&) = delete;
};
```

By disabling the copy constructors the previously shown error can no longer occur. It is sort of odd though. It is clearly okay to `memcpy()` the ArrayHeader, that's what it is designed for. Yet disabling the copy constructors and telling the C++ compiler that it is not okay reduces the amount of potential bugs. Strange language.

And that's basically all I'm going to say about the ArrayHeader for now. If you want to see more details checkout the source in [Phantasy Engine](https://github.com/PetorSFZ/PhantasyEngine).

## GameStateHeader

Now that we have the basic building block in the form of `ArrayHeader` we can design our single-chunk game state. We use a similar pattern to `ArrayHeader` where we have a header which operates on memory adjacent to it.

But first up, the memory layout. The memory layout of a game state is something as follows:

```cpp
// No struct like this exists of course. The amount of memory needed and
// all offsets are calculated dynamically when the game state is initially
// created.
struct GameState {
  // The game state header. Contains all logic for accessing stuff inside
  // the game state. Similar to ArrayHeader but more complex. Also contains
  // offsets to all "members in GameState" except for specific singletons
  // and component arrays, which need to be looked up in the registries.
  GameStateHeader header;
  
  // Registry with information about offsets to and size of each singleton
  ArrayHeader singletonRegistryHeader;
  SingletonRegistryEntry singletonRegistryEntries[NUM_SINGLETONS];
  
  // Singletons are placed here
  Singleton0 singleton0;
  Singleton1 singleton1;
  // ...
  SingletonN singletonN;
  
  // Registry with information about offsets to the ArrayHeader for each
  // component type
  ArrayHeader componentRegistryHeader;
  ComponentRegistryEntry componentRegistryEntries[NUM_COMPONENT_TYPES];
  
  // Array with all components masks for all possible entities
  ArrayHeader componentMasksHeader;
  uint64_t componentMasks[MAX_NUM_ENTITITES];
  
  // Array with all generations for all possible entities
  ArrayHeader generationsHeader;
  uint8_t generations[MAX_NUM_ENTITIES];
  
  // All components
  ArrayHeader componentType0Header;
  Component0 component0s[MAX_NUM_ENTITIES];
  ArrayHeader componentType1Header;
  Component1 component1s[MAX_NUM_ENTITIES];
  // ...
  ArrayHeader componentTypeNHeader;
  ComponentN componentNs[MAX_NUM_ENTITIES];
};
```

It is worth noting that every single thing (i.e. member) in the example above is 32-byte aligned. ArrayHeader's being naturally 32-byte aligned helps, but there is also some manual padding after arrays to ensure the next thing is also 32-byte aligned.

Offsets to specific singletons or component arrays need to be looked up in singleton or component registries respectively. So a single extra indirection. Technically we could get rid of this indirection by setting a maximum number of component and singleton types (which we technically already do, but whatever) and storing that number of entries directly in the `GameStateHeader`, but I honestly don't see that much need for it.

At this point we can take a quick look at (a simplified version of) the `GameStateHeader` struct:

```cpp
struct GameStateHeader {
  // Magic number in beginning of the game state. Should spell out "PHESTATE" if viewed in a hex
  // editor. Can be used to check if a binary file seems to be a game state dumped to file.
  // See: https://en.wikipedia.org/wiki/File_format#Magic_number
  uint64_t magicNumber;

  // The version of the game state, this number should increment each time a change is made to
  // the data layout of the system.
  uint64_t gameStateVersion;

  // The size of the game state in bytes. This is the number of bytes to copy if you want to copy
  // the entire state using memcpy(). E.g. "memcpy(dst, stateHeader, stateHeader->stateSizeBytes)".
  uint64_t stateSizeBytes;

  // The number of singleton structs in the game state.
  uint32_t numSingletons;

  // The number of component types in the ECS system. This includes data-less flags, such as the
  // first (0th) ComponentMask bit which is reserved for whether an entity is active or not.
  uint32_t numComponentTypes;
  
  // The maximum number of entities allowed in the ECS system.
  uint32_t maxNumEntities;

  // The current number of entities in this ECS system. It is NOT safe to use this as the upper
  // bound when iterating over all entities as the currently existing entities are not guaranteed
  // to be contiguously packed.
  uint32_t currentNumEntities;

  // Offset in bytes to the ArrayHeader of SingletonRegistryEntry which in turn contains the
  // offsets to the singleton structs, and the sizes of them.
  uint32_t offsetSingletonRegistry;

  // Offset in bytes to the ArrayHeader of ComponentRegistryEntry which in turn contains the
  // offsets to the ArrayHeaders for the various component types
  uint32_t offsetComponentRegistry;

  // Offset in bytes to the ArrayHeader of free entity ids (uint32_t)
  uint32_t offsetFreeEntityIdsList;

  // Offset in bytes to the ArrayHeader of ComponentMask, each entity is its own index into this
  // array of masks.
  uint32_t offsetComponentMasks;

  // Offset in bytes to the ArrayHeader of entity generations (uint8_t)
  uint32_t offsetEntityGenerationsList;

  // Unused padding to ensure header is 32-byte aligned.
  uint32_t ___PADDING_UNUSED___[1];

  /* ... */
};
```

Mostly this is fairly unsurprising, just a lot of offsets as expected. But there is something interesting in the beginning of the struct. A `magicNumber` and  `gameStateVersion`. These are used to determine if a random file on disk is a Phantasy Engine game state, and which version it is. The version is incremented manually each time we make changes to the memory layout of the game state. Essentially, this whole thing is kind of like operating on a binary file directly.

And this is pretty much it. We have some nice helper methods in the `GameStateHeader` to make it easier to access everything in it, but it is not too complicated if you understand how the `ArrayHeader` works. Look up the [source](https://github.com/PetorSFZ/PhantasyEngine/tree/master/engine/include/ph/state) if you are interested in more details.

## Conclusion

Overall I am pretty happy with how Phantasy Engine's game state has turned out. It has been a bit of a rocky road, if you look up the git history there is an embarrassing amount of stupid errors made along the way. But the end result is really neat, having everything in a single chunk of memory has already turned out useful for serialization/deserialization purposes. And I can already imagine other cool applications for it.

Unlike the last post I think I covered most of the technical details I wanted to cover, but as tradeoff I think it got way too long. Oh well, it's hard to be brief when trying to explain technical details. Hopefully you, the reader, has gotten something out of it. Maybe learned some new C++ trick, or maybe realized something you had never thought about. I know at least I did a couple of times when I implemented this myself.

That is everything for me. Don't forget to [follow me on twitter](https://twitter.com/petorsfz) for more updates!

