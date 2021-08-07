---
title: "evol-scripting"
description: "The evol game engine's scripting module."
date: 2021-07-31T13:25:00Z

categories: ["Project"]
tags: ["Game Engine"]

toc: true

hiddenFromHomePage: true

draft: false
---

The scripting module was written in a way that makes it as flexible and as 
accessible as possible for all modules that might need it. First of all, 
“Game World” systems were created to allow the running of per-frame update 
functions that could be written in scripts. This meant that systems will only 
need to run for entities that have those script functions without needing to 
iterate over all entities to check.

To make extending this functionality substantially simpler, a lot of 
preprocessor macros were added to that file so that adding a new system for a 
new script function is just a matter of adding an extra line to the file and it 
should handle everything else. For example, adding an `on_fixedupdate` function 
to the scripting module is as simple as changing this:
```c
#define SCRIPT_CALLBACK_FUNCTIONS() \
  SCRIPT_OP(on_init)                \
  SCRIPT_OP(on_update)
```
to this:

```c
#define SCRIPT_CALLBACK_FUNCTIONS() \
  SCRIPT_OP(on_init)                \
  SCRIPT_OP(on_update)              \
  SCRIPT_OP(on_fixedupdate)
```

However, after a few runs, it was noticeable that having the scripts be run in 
a system meant that it had a few limitations when it came to editing, removing 
or adding new entities into the scene. Since the ECS world uses the archetype 
approach for the storage of components, the addition and removal of components 
to an entity means that this entity will belong to another archetype and thus 
needs to be moved to another table. Because of that, the addition of removal of 
components to entities is prohibited in the phase where systems are ran as 
these additions might change the tables on which the system is iterating. 
That’s why we came up with the concept of a task.

A task is a lot like a system, it takes a component signature, matches with all 
entities that fulfill that signature and iterates over all of them. The main 
difference, though, is that a task is not run in the system phase of the ECS 
world; tasks are run when the ECS is in an idle state and thus is open to big 
changes in its tables. This allowed us to have more world-changing operations 
that can run in the scripts, like the loading of prefabs.

Then, lots of functions were exposed from the scripting module to allow the 
addition of script components to a “Game World” entity without needing to get 
into much detail from other modules. This way, a script can be added to an 
entity as follows:
```c
ScriptHandle playerScript = Script->new("PlayerScript", loadedScript);
Script->addToEntity(player, playerScript);
```

Then came the problem of exposing C structs and functions in a not-too-verbose
way and making this operation happen per module at runtime. Luckily, we found 
orangeduck's [LuaAutoC](https://github.com/orangeduck/LuaAutoC) which did just
that; it allowed the quick and automatic exposure of functions and structs from
C to Lua with minimum friction. Unfortunately, to make the API less verbose,
most of the library's API was preprocessor-based and wouldn't work across DLL
boundaries. To fix that, we made a [fork](https://github.com/evol3d/luaautoc)
in which we made major API changes to allow this functionality to work using
only functions and thus be able to use from all the modules. This was then
exposed through the `ScriptInterface` namespace which is in the scripting
module. This namespace’s main purpose was to allow modules to register 
whatever they want to expose to modules, like: types, functions, structs, ..etc. 
The `ScriptInterface` consisted of the following operations:

| Function      | Description                                                                                                                                                                                                                                    |
|---------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `addType`     | Creates a Lua type from a given type name and a given type size                                                                                                                                                                                |
| `getType`     | Get the identifier of a type using a given name                                                                                                                                                                                                |
| `addFunction` | Expose a function to Lua using its name, its reference, and a list of the arguments that it expects                                                                                                                                            |
| `addStruct`   | Expose a struct to Lua using its typename, its size, and a list of its parameters’ names and their types                                                                                                                                       |
| `loadAPI`     | Takes a file path that points to a Lua file that contains the description of the API that the module wants to expose. (This gives modules the ability to tweak their scripting APIs freely without needing the Scripting Module to intervene.) |

While the `loadAPI(..)` function might seem redundant, it was added for a very 
specific reason. The `addFunction(..)` function does indeed expose module 
functions, but in its own way. For example, exposing a function by doing this:
```c
GameObject ev_sceneloader_loadprefab(string path);
ScriptInterface->addFunction(ev_sceneloader_loadprefab, "ev_sceneloader_loadprefab", gameObjType, 1, (ScriptType[]){stringType});
```
will expose it to Lua in a way that makes it callable by doing the following:
```c
prefab = C('ev_sceneloader_loadprefab', path)
```

That grows inconvenient quite fast, however, by having an API-defining Lua 
file, the Lua file can contain something like:
```lua
function loadPrefab(path)
  return C('ev_sceneloader_loadprefab', path)
end
```
and the previous line can be written as:
```lua
prefab = loadPrefab(path)
```

After all of that was done, it was time for the module to move on from being a 
singleton and start to become more instantiate-able. This is where a script 
context comes into play. A script context is a type that we defined that 
contained some info about itself and the Lua VM that it contains. The fact that 
each context had its own VM had solid reasons; the most important one is that 
Lua has everything as global by default. That meant that name collisions were a 
lot more probable and that objects would be able to access other objects from 
other scenes; that’s why each context had its own VM.

However, having every context have its own VM means that types and functions 
will need to be registered for each of these contexts separately. So we decided 
to add script context creation callback functions, functions that are called 
and passed the context whenever a new context is created. This allows modules 
that want to register their own types to just create that callback and pass it 
to the scripting module so that it can store it. Then, whenever a new context 
is created, the scripting module calls all the callback functions that it has 
stored from different modules and thus rebuilds the API for that context.

One last thing that remained was to make the debugging of scripts a bit more 
systematic. That’s why we integrated 
[luadbg](https://github.com/slembcke/debugger.lua) into the scripting module. 
luadbg helped us by making a debugger kick in whenever occurs in a script (or 
whenever we call a function that signals the debugger) so that we can inspect 
the state of the Lua VM at that specific point in time.

{{< figure src="/evol/images/scripting/debugger.png" alt="Script Debugging Diagram" title="Figure 1: luadbg in action" >}}
