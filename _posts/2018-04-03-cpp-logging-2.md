---
layout: post
title: "C++ Logging (Part 2)"
categories: development
comments: true
---

# C++ Logging (Part 2)

Hi again! This second part took a bit longer than expected. I got a bit distracted by all the great games that have been released recently, specifically [Celeste](http://store.steampowered.com/app/504230/Celeste/) and [Iconoclasts](http://store.steampowered.com/app/393520/Iconoclasts/). But now I have finally gotten around to implementing the actual logger.

## The interface

The logger have the following interface, as shown in the previous blog post:

~~~cpp
class LoggingInterface {
public:
    virtual ~LoggingInterface() {}
    virtual void log(
        const char* file,
        int line,
        LogLevel level,
        const char* tag,
        const char* format,
        ...) = 0;
};
~~~

The default logger just prints all the information using `printf()`. But for my engine ([Phantasy Engine](https://github.com/PhantasyEngine)) I don't want to be dependent on printing to an OS terminal. For example, on Window I don't want to create a CMD window at all for a GUI application. Therefore my logger implementation will log directly to an [ImGui](https://github.com/ocornut/imgui) window.

## The implementation

Since ImGui does not have a state, it becomes necessary to store the messages so we can retrieve them later. Each message is stored using the following struct:

~~~cpp
struct TerminalMessageItem {
    StackString64 file;
    int32_t lineNumber;
    time_t timestamp;
    LogLevel level;
    StackString32 tag;
    StackString256 message;
};
~~~

[`StackString`](https://github.com/PetorSFZ/sfzCore/blob/master/include/sfz/strings/StackString.hpp) is a string class in [sfzCore](https://github.com/PetorSFZ/sfzCore), it essentially looks something like this:

~~~cpp
template<size_t N>
struct StackString {
    char str[N];
};
~~~

These messages are stored in a [ring buffer](https://en.wikipedia.org/wiki/Circular_buffer), which ensure that memory will not grow uncontrollably if a lot of log messages are generated. The logger class interface looks something like this:

~~~cpp
class TerminalLogger : public LoggingInterface {
public:
    // Accessors for logged messages
	uint32_t numMessages() const;
	const TerminalMessageItem& getMessage(uint32_t index) const;
	
    // Implementation of log function
    void log(
        const char* file,
        int line,
        LogLevel level,
        const char* tag,
        const char* format,
        ...) override final;
private:
    RingBuffer<TerminalMessageItem> mMessages;
};
~~~

Aside from the implementation of the log function, it is also possible to access the logged messages. This is necessary for the ImGui code.

## The result

I won't show the ImGui code in this blogpost as it is a bit ugly and still evolving, if you want to take a look it is currently available in [this file](https://github.com/PhantasyEngine/PhantasyEngine-Core/blob/147524828bed350202d9cc8603d577903a41a78d/src/ph/game_loop/DefaultGameUpdateable.cpp#L483-L589). The result currently looks like this:

![](/assets/posts/2018-04-03-cpp-logging-2/result.png)

It turned out that printing all the information in each message gets quite messy. For this reason only the tag and the messsage itself is explicitly printed out. The log level is indicated by the color of the message, warnings and errors have strong colors that immediately stand out. The additional information stored in the message can be viewed in a tooltip by hovering over the message:

![](/assets/posts/2018-04-03-cpp-logging-2/hover.png)

Messages can be filtered in two ways with the current UI. First, a minimum log level can be set (defaults to `INFO_NOISY`, the lowest level). All messages with a less severe log level than this will be filtered out. Secondly, messages can be filtered by tags:

![](/assets/posts/2018-04-03-cpp-logging-2/filter.gif)

There are of course many more improvements that can be done to this logging system, both in the UI and in the backend. Something I'm planning to add in the future is the ability to also log the messages to a log file in addition to the in-game terminal. But for now the current setup is good enough.