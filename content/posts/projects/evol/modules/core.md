---
title: "evol-core"
description: "The core of the evol framework used to build the evol game engine."
date: 2021-07-31T13:25:00Z

categories: ["Project"]
tags: ["Game Engine"]

hiddenFromHomePage: true

toc: true

draft: false
---

First of all, comes the core. The core was implemented in a way that ensures 
that all core functionalities are provided to the modules without the need for 
complex dependencies. The first thing that needed to be done was to make the 
modules loadable/unloadable at runtime. While multiple approaches were tried 
out at the beginning, the most stable one was the use of dynamically loaded 
libraries (DLLs). The main advantage of using DLLs is the fact that the 
load/unload operation is handled primarily by the operating system. The fact 
that all modern operating systems implement reference counting for DLLs also 
made their usage easier as that is one feature that was needed. While DLLs are 
similarly implemented, each operating system might have its own format to 
represent it, for example: Microsoft Windows uses (PE) and Linux uses (ELF64). 
Luckily, each operating system has its own API which exposes a set of functions 
that are similar in functionality throughout platforms. With that in mind, all 
that needed to be done was to wrap the OS-specific calls in some 
platform-agnostic functions that lifted the burden of caring for the operating 
system off the modules.

| Function              | Description                                                                                        |
|-----------------------|----------------------------------------------------------------------------------------------------|
| `ev_module_open`      | Opens module (given its path) and increments its reference count.                                  |
| `ev_module_close`     | Returns handle of a module (given its path) if it is already loaded in memory.                     |
| `ev_module_gethandle` | Closes a module (given its handle) and decrements its reference count.                             |
| `ev_module_getfn`     | Gets a function that is exposed by a module (given the module handle and the name of the function) |
| `ev_module_getvar`    | Gets a variable that is exposed by a module (given the module handle and the name of the variable) |


After that, the next step was to provide the modules with a way to run custom 
behavior whenever they’re loaded or unloaded. To do that, the 
constructor/destructor functionality was implemented for those modules. Some 
research was needed to identify the best approach to implement that feature. 
Luckily, we found that both the ELF64 and PE standards implement ways for the 
DLLs to run specific functions on load/unload. The ELF64 uses the .init and 
.fini header sections and PE uses an optional DLLMain function that runs both 
on initialization and de-initialization. After wrapping these functionalities in 
our own functions, the following was the minimal format for a module:
```c
// testmod/mod.c
EV_CONSTRUCTOR
{
  ev_log_info("Module Loaded Successfully");
}

EV_DESTRUCTOR
{
  ev_log_info("Module Unloaded Successfully");
}
```

With that layout, the code written in the `EV_CONSTRUCTOR` is run as soon as a 
module is initially loaded into memory (its reference count was zero and now it 
is one) and the `EV_DESTRUCTOR` is run when no other module needs it anymore 
(its reference count was just decremented to reach zero).

Before adding more helpful module-managing functionalities, more utility types 
and features were needed to be added to the core so that it can expose them to 
the other modules and also use them for the module maintaining process.

## Dynamic Arrays

Since we’re using C, the first thing that we needed was a generic dynamic 
array. To do that, we built the `vec` single-header library. While building 
this library, there were multiple objectives that we had for it so that its 
usage was as painless as possible. Those objectives were:
- Should be usable in the same scenarios that a normal statically-sized array would be used in
- Arrays should be index-able in the same way as statically-sized arrays (using square brackets)
- Array elements should be allowed to have their own destructors to avoid dangling pointers
- Array elements can have their own copy function specified to allow deep copies when needed

The first two objectives defined the direction in which the vector structure 
would go. While a dynamically sized array needs to have a lot of its meta-data 
bundled with it, having the meta-data with the vector meant that there would be 
a distinction between it and the normal arrays. To solve this problem, we 
decided to store the vector’s meta-data right before it. This way, while the 
parts using the vector will see it as a normal array, the library functions 
will be aware that a specific length of memory that lies right before that 
vector are its actual meta-data and we’ll be able to retrieve that data easily.

We ended up trying out two approaches:
- The intuitive one
```c
// Intuitive Approach
struct {
  size_t len; // Length of the vector
  void *data; // Actual data inside the array
} vec;
// The vector is initialized with the size of the element so that it can be more generic
vec int_vec = vec_init(sizeof(int));
int elem = 1;
vec_push(int_vec, &elem);
// The length is directly accessible
int_vec.len;
// The vector can't be indexed directly; the data member is the one that is indexed.
((int*)int_vec.data)[1];
```

- The better one
```c
// Our (Better) Approach
#define vec(T) T*
// The use of preprocessor macros allows the use of types to initialize vectors instead of their size
vec(int) int_vec = vec_init(int);
int elem = 1;
vec_push(&int_vec, &elem);
// The length is not directly accessible but a helper function gets it easily
vec_len(int_vec);
// The vector can be indexed directly as it points to the start of the array without the interference of any of its meta-data
int_vec[1];
```
Due to the simpler and more readable nature of our approach, it was favored 
over the intuitive one that would make the usage of vectors more of a chore.

## Hashmaps

After the dynamic array, we needed another data structure that we’ll use 
countless times throughout the project, a hash table. At first, we tried 
implementing our own hashmap, however, since this was at the beginning of the 
project and we didn't have much experience with C’s advanced features, we 
couldn't do it in a way that is generic enough to be usable in arbitrary 
places. That’s why we changed our goal from developing our own hashmap to 
finding one that is generic and efficient and using it instead. That’s when we 
found tidwall's [hashmap.c](https://github.com/tidwall/hashmap.c).

After going through the library, making sure it does everything we need it to, 
and understanding how it does it, we had a small problem with it; its API was 
(in our opinion) very unclean. While that would seem like a minor problem that 
can be ignored, the API’s problems were mainly:
- The hashmap treats keys and values as a single object. Thus, having a hashmap 
  for a specific key-value pair would require us to define our own type that 
  combines this pair and defining our own custom hashing and comparison 
  functions so that the hashmap doesn't knows how to interact with that object.
- The hashmap has no concept of destructors. While that wouldn't be a problem 
  with plain objects that don’t have any references to other data, as soon as a 
  hashmap element has a reference, removing that element from the map will most 
  probably result in a memory leak (unless treated specifically.)

These two problems, while not critical, would put us in a place where it’s too 
risky to use the hashmap without having them in mind. Luckily, both of those 
problems could be fixed and the API could be made way better with the help of 
the C preprocessor without sacrificing any of its efficiency or genericness.

Now that the core is able to provide the most basic utilities, it needs to 
provide a way for modules to communicate by sending notifications when 
important events occur. For that feature, an event system is needed.

## Event System

One of the major challenges of having an event system written in C is the fact 
that events are, by nature, very dependent on polymorphism. For example, there 
are generic events like: Window Event, Mouse Event, Key Event, ..etc and more 
detailed events like: KeyDown Event, MouseMoved Event, ..etc. Another problem 
is that event dispatches are supposed to work with all events that may have 
different sizes and thus require a way for the compiler to know what to do with 
each type. Unfortunately, doing so will make the simple act of defining a new 
event type verbose enough to be unreadable in the long run.

Luckily, this was found to be a bit more manageable than initially expected. 
First of all, for the events to be passable through functions, they’d need to 
have the same size, which is impractical. What would have a constant size is 
each event’s reference. After that comes the part of actually knowing what kind 
of event it is as all references look the same. This was fixed by adding an 
“event type” field in the event that will be the first few bytes at the memory 
address the event references. 

As for supporting some degree of polymorphism, events were split into “Primary” 
and “Secondary” events. This way, the event type field could simply be made to 
be a single 32-bit value of which the first 16 bits are the primary type of the 
event and the other 16 bits are the secondary event which gives more details 
about the event itself.

Now that events are split into two degrees (Primary and Secondary), it would be 
nice if secondary events had the same parameters that their parent “Primary” 
event has without the need to specify those fields manually for each event. 
While the way to do that would be trivial to find in an object-oriented 
language, doing so in C was not that obvious, until we noticed one of the 
lesser known features of the C11 language standard; it’s called “anonymous 
structs” and it allows structs to have other (anonymous) structs of which they 
inherit the fields without explicitly exposing the fact that they have that 
struct inside them.

Now that the structure of the events is finished, the next step is to see how 
the event-system will dispatch said events. The main idea was that there will 
be a list of events that is filled throughout the frame and then, at a specific 
stage, all those events will be dispatched to their listeners and then the list 
will be cleared. An important step was to ensure thread safety in that event 
list so that modules can dispatch events without needing to rely on a single 
event-dispatching thread. This was easily done by using a mutex that ensures 
exclusive access when adding events to that list.

A problem that we didn't notice unless it started affecting us is that event 
handlers can dispatch other events. This means that new unhandled events will 
be added to the list right before it is cleared. This was fixed by building a 
double-buffered vector; a buffer for reading and a buffer for writing. This 
way, the two buffers are to be swapped each frame so that the one that was 
being written to is the one being currently read for event handling while new 
events are being added to the write buffer.

As for event listeners, they were given some flexibility in what they can 
listen to. Due to the fact that the event type contains two values (the 
primary type and the secondary type), the listeners can either choose to 
listen for a specific secondary type or to listen for a primary type and 
therefore listen to all the secondary events that have that type in as their 
primary type.

## Module Manager

Next in line is the module manager. The module manager’s main responsibility is 
to keep track of the modules that are available for the framework to load on 
demand. This helps when modules need to load other modules without knowing 
their exact path in the file-system. The first thing that this module manager 
does is that it takes a module directory that it scans to identify all the 
available modules, their names and their categories. After that, whenever a 
module needs to be loaded, it can be referenced using its name or category. 
This means that instead of writing
```c
ev_module_open("./modules/bullet-physics.evmod")
```
this can be written:
```c
evol_loadmodule("physics")
```
while maintaining the intended behaviour.

## Configuration File

To allow tune-able parameters in the framework and in modules, a configuration 
file was needed. That is why a configuration loader was created. The 
configuration loader’s goals were pretty simple: we needed a file to contain 
some variables that the modules could look up at runtime. As for the format of 
that configuration file, we settled on Lua since we were already planning on 
having the scripting engine be made in Lua and thus this would help us reuse 
some of the utilities we will create for interacting with Lua.

That was a good time for us to start creating our Lua utility functions. We 
created several functions that, in our opinion, should help us interact with 
almost any arbitrary Lua file without the need to deal with the low level 
details of the Lua stack. Those functions were:

| Function--             | Description                                                                                                                              |
|------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| `ev_lua_newState`      | Creates a new Lua VM and loads the standard libraries into it                                                                            |
| `ev_lua_destroyState`  | Destroys given Lua VM                                                                                                                    |
| `ev_lua_loadfile`      | Loads a Lua file from the file-system and pushes it to the top of the given Lua VM’s stack                                                |
| `ev_lua_runloadedfile` | Executes the file that is on top of the Lua stack (the last file loaded by `ev_lua_loadfile`)                                              |
| `ev_lua_runfile`       | Loads and executes that contents of a file (equivalent to calling `ev_lua_loadfile` followed by `ev_lua_runloadedfile`)                      |
| `ev_lua_getstring`     | Takes a Lua variable name, finds it in the Lua VM, checks to see if it is a string, then returns it.                                     |
| `ev_lua_getint`        | Takes a Lua variable name, finds it in the Lua VM, checks to see if it is an integer, then returns it.                                   |
| `ev_lua_getdouble`     | Takes a Lua variable name, finds it in the Lua VM, checks to see if it is a double precision floating point number, then returns it.     |
| `ev_lua_runstring`     | Takes a Lua code string and executes it                                                                                                  |
| `ev_lua_callfn`        | Takes a Lua function name, its parameters, and variables in which the return values should be written, finds that function and calls it. |


## Threads & Thread-pools

One of the most important utilities needed by any software is the ability to 
use multiple threads to make use of the concurrency that is built into modern 
CPUs. At first, it seemed as if adding thread utilities would be a breeze since 
the C11 language standard already provided concurrency primitives in the 
standard libraries. However, due to the fact that C11 had it as an optional 
feature, the Visual Studio compiler (MSVC) didn't feel the need to provide it 
and only provided the primitives that are already exposed through the Windows 
API.

Luckily, the usage of Windows API concurrency primitives was quite similar to 
their Unix counterparts and thus wrapping them into the same API was, despite 
being a chore, pretty trivial. For the wrapping style, we decided to use the 
pthreads API that is already provided in Unix systems as a reference. This way, 
we only needed to wrap the Windows API without needing to make any changes on 
non-Windows systems.

Now that we had cross-platform concurrency primitives, we needed a way to 
remove the overhead of creating those primitives for systems that will 
consistently need the creation of those primitives periodically. That is why we 
implemented a threadpool. A threadpool is quite simple in concept; it is simply 
a list of threads that are already initialized and are waiting to be used. Work 
is submitted to that list by simply adding it to a list of job requests that 
threads check to see if there is anything for them to do. While stopping here 
would be quite reasonable, having the threads regularly check that work list 
would mean wasting a lot of the CPU time doing nothing but checking a list that 
would probably have nothing in it most of the time. That’s why we added signals 
to the mix, by making the threads wait on signals and having those signals be 
dispatched whenever a new job is added to the job list, threads are put into a 
sleep state and don’t take any CPU time when there is no work to be done.


## Dynamic Strings

The fact that we’re using C meant that we had to deal with one of its more 
daunting problems; the fact that C’s strings are extremely unsafe. To deal with 
that, we had to develop our very own safe string library. In that library we 
used the experience gathered throughout all the previous steps to try making a 
string library that is not difficult to use. We had a few goals that we needed 
to meet so that we could consider this library a success:
- The library’s strings should be usable by functions that expect normal C 
  strings.
- A string’s length should be cached as getting a string’s length is a common 
  operations
- Strings should not need to be duplicated or moved when they are needed for 
  simple operations

The first two objectives looked similar to those of the vector type that we 
already did. That’s when we knew that our string will basically be a 
specialization of the vector made only for the characters. This means that 
we’ll have the important meta-data stored before the string without the user 
ever needing to deal with it directly. The first requirement, though, meant 
that we wouldn't be able to get rid of the null terminator and that we’d need 
to keep it at the end of every string so that existing string functions can 
work with our strings. We ended up with the following API:

| Function                | Description                                                                                                                                               |
|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| `evstring_new`          | Creates an evstring from a given C string                                                                                                                 |
| `evstring_clone`        | Creates an evstring from a given existing evstring                                                                                                        |
| `evstring_free`         | Destroys the given evstring                                                                                                                               |
| `evstring_len`          | Returns the length of the given evstring                                                                                                                  |
| `evstring_setlen`       | Sets the length of the given evstring to a given value                                                                                                    |
| `evstring_cmp`          | Checks the equality of two strings                                                                                                                        |
| `evstring_pushchar`     | Appends a given character to the given evstring (dynamically resizing it as needed)                                                                       |
| `evstring_pushstr`      | Appends a given string to the given evstring (dynamically resizing it as needed)                                                                          |
| `evstring_pushfmt`      | Appends a new string denoted by a given format string and several values to the end of the given evstring (dynamically resizing it as needed)             |
| `evstring_getspace`     | Returns the allocated space at the end of the given evstring that contains no data. (Size of data that can be pushed without the need for a reallocation) |
| `evstring_addspace`     | Adds more space to the given evstring so that more data can be pushed without needing reallocation.                                                       |
| `evstring_clear`        | Clears the data that’s inside the given evstring and sets its length to zero.                                                                             |
| `evstring_newfmt`       | Creates a new evstring from the given format string and the following variables that contain data referenced in the format string.                        |
| `evstring_findfirst`    | Returns the first occurrence of a given query string inside the given evstring                                                                            |
| `evstring_replacefirst` | Replaces the first occurrence of a given query string inside the given evstring with the given replacement string (dynamically resizing it as needed)     |
| `evstring_findfirst_ch` | Finds the index of the first occurrence of a character inside the given evstring                                                                          |
| `evstring_findlast_ch`  | Finds the index of the last occurrence of a character inside the given evstring                                                                           |


While the already implemented functions already provide a full set of utilities 
that can deal with almost all the operations that we might need to do with a 
string, it still doesn't fix the problem of needing to copy strings for a lot 
of operations that don’t require such an inefficient operation (like slicing 
the string). That’s why we came up with the idea of a string reference. A 
string reference contains exactly three values: a reference to the evstring 
it’s referencing, the span of the reference, the offset of the reference inside 
the evstring. This way, a lot of string duplications could be prevented. The 
following functions were added to the evstring API:

| Function            | Description                                                                                                  |
|---------------------|--------------------------------------------------------------------------------------------------------------|
| `evstring_ref`      | Creates an evstring reference from the given evstring                                                        |
| `evstring_refclone` | Creates an evstring from a given evstring reference.                                                         |
| `evstring_slice`    | Creates an evstring reference that denotes a slice of the given evstring.                                    |
| `evstring_refpush`  | Appends the contents of a given evstring reference to the given evstring (dynamically resizing it as needed) |


## JSON Parser
Since we already knew that we’ll eventually have our scene format and project 
configuration be in JSON, we decided early on to write our own JSON parser that 
makes use of our evstring so that it can reduce its memory footprint 
substantially. We started by writing a JSON tokenizer; the idea was simple: 
iterate through the JSON file, identify tokens, add them to a token list along 
with their types. This token list benefited heavily from the evstring 
references as token data was merely a string reference that points to the 
actual data that is in the original JSON string.

Now that we have all the tokens ready for interpretation, we start iterating 
through those tokens and adding them to a hashmap that has the path to each 
JSON value as the key to ease the process of retrieving the values from it 
later. This meant that for a simple JSON query, we could write the following:
```c
evjs_get(json_context, "application.version.major")
// or
evjs_get(json_context, "scenes[2].name")
```

## Module Utilities

Now that a lot of features are available for modules to benefit from, we needed 
a way to automate some of the common operations for modules, like:
- Creation and exposing namespaces
- Definition and loading configuration variables from the configuration file
- Definition and exposing events that other modules can use

For that, several files were standardized for a module and each of those files 
was created for the purpose of containing a single set of operations. For 
example, there was the “evmod.configvars” file; this file’s main purpose was to 
define configuration variables that the module was expecting from the 
configuration file along with their types and default values. The file’s format 
was as follows:
```c
// Window Title
EV_CONFIG_VAR(window_title, STRING, "Default Title")
// Physics Visualization flag
EV_CONFIG_VAR(visualize_physics, U8, 0)
```

This way, whenever one of those configuration variables is added to the 
configuration file, this module will be able to access it. Otherwise, it will 
simply use the default value that it specified. The fact that the configuration 
variable is defined in a separate file and through the use of preprocessor 
macros, the loading of the configuration variable was automated so that the 
value is automatically loaded whenever the module is initialized. This way, the 
module can easily start using the configuration variable as if it was an 
already defined global variable without ever needing to load it explicitly from 
the configuration loader.

The same goes for another file called “evmod.events”. The purpose of this file 
is to serve as the place in which a module’s events are defined. The file’s 
structures looks something like:
```c
// Primary Events
PRIMARY(MouseEvent, { GenericIOEvent; })
PRIMARY(WindowEvent, { GenericIOEvent; })

// Secondary Window Events
SECONDARY(WindowEvent, WindowResizedEvent, { U32 newWidth; U32 newHeight; })

// Secondary Mouse Events
SECONDARY(MouseEvent, MouseMovedEvent, { MousePosition position; })
SECONDARY(MouseEvent, MouseButtonPressedEvent, { GenericMouseButtonEvent; })
```

There are two other files that are called “evmod.types” and “evmod.namespaces” 
that do the same to expose module-specific types and namespaces that the module 
exposes.
