---
title: "evol"
description: "My very first game engine"
date: 2021-07-31T13:25:00Z

categories: ["Project"]
tags: ["Game Engine"]

toc: true

featuredImagePreview: "/evol/images/logo.png"

draft: false
---

A game engine built using C.
<!--more-->

## Introduction
Being bored and having to pick a capstone project for university's last year, I
thought to myself: "Why not build a game engine?" I knew from the start that
this would definitely be a difficult task that I will most probably not be able
to fully tackle in time, but I was too bored to care :D. So I went ahead,
gathered a team of three (including me) and set out to build our very first game
engine.

## Objectives

The engine had some objectives, some of which were met and others that weren't.
Those objectives were:

- **Be as modular as possible**: Due to the fact that the engine was
    expected to keep on evolving, it was decided that it'd be better to have
    it be modular enough for subsystems to be easily replaced without
    needing to compile the entire engine whenever something is changed in
    one of its modules.
- **Be cross-platform**: Cross-platform is such a loose term, when you thing
    about it; all we did was support both Windows and Linux and we got to
    call the engine "cross-platform", so I'd call that a win.
- **Be mostly written in C**: uhmmm... There is actually no specific reason
    for wanting to write a game engine in C; we simply settled on doing that
    because it seemed more challenging and thus would give us more style
    points. That and the fact that writing pure C actually helped us learn a
    lot more about what the limitations of the language are and how they can
    be worked around.

## Architecture

For the engine, we decided to use some sort of a hybrid architecture which lies
somewhere between a plugin architecture and a layered architecture. This would
provide us the flexibility of a plugin architecture with the stable structure 
of a layered architecture. For more details about the architecture, you can
checkout this [post]({{<ref "/posts/projects/evol/modules/architecture.md">}})

## Implementation
While I would like to keep on babbling on what was done throughout the engine's
implementation phase, it would be hard to do that without making the engine a
two-hour read. That's why I'll just write a couple of sentences per section and
provide links for other pages that will have a more verbose description of each
section.

### Core
Since we settled on a variant of the plugin architecture, we needed some of a
framework in which to attach the different modules This framework is the
engine's core and can be found in [this](https://github.com/evol3d/evol)
repository. The core provided all necessary utilities that the modules might
need to operate, like functions to:
- Query for existence of modules
- Load/Unload modules
- Ask modules for functions that they expose

It also provided some QoL features, like:
- Module constructors/destructors
- Common data structures (vectors, hashmaps)
- Unified configuration file

However, the most crucial features that were provided in the core were:
- The event system
- The global store (which provides all the modules with a place to store data
  they want to be easily accessible from other modules)

For more details, feel free to look [here]({{<ref "/posts/projects/evol/modules/core.md">}})

### Scripting
For the scripting language, we decided on using Lua. That's why we created a
scripting module that uses the LuaJIT compiler to run scripts on game entities.
`ScriptComponent`s, that can be attached to game entities, were created. Regular
interval functions were also created to facilitate executing the scripting
behaviour at different rates, for example:
- `on_init()`: Runs on entity initialization
- `on_update()`: Runs once per frame
- `on_fixedupdate()`: Runs with a fixed rate of 60 times per second

Moreover, functions were also exposed through the module's namespaces to allow 
seamless exposure of module-specific functions and structs to the Lua side. This
was done using a heavily mutilated [fork](https://github.com/evol3d/luaautoc) of
[LuaAutoC](https://github.com/orangeduck/luaautoc).

To allow easier script debugging, a Lua
[debugger](https://github.com/slembcke/debugger.lua) was integrated into the 
module. This made script debugging a lot more like C debugging using gdb and, 
in turn, improved the workflow substantially.

More details and code snippets can be found [here]({{<ref "/posts/projects/evol/modules/scripting.md">}})

### Physics
For the physics, we decided to use the Bullet Physics library. So the physics
module was mostly a wrapper over bullet. We also added collision callbacks so
that collided objects can attach scripts that execute custom behaviour when
colliding. Functions were exposed to the Scripting API for:
- Setting/Getting Velocities
- Adding Forces
- Creation of ray-casts
- ..etc.

To improve the workflow a bit, a physics debugger was created that can visualize
the collision shapes of rigid bodies and thus can give more insight about how
the rigid bodies interact with each other that wouldn't be obvious in the actual
game view.

{{< figure src="/evol/images/physics/phys_vis.png" alt="Physics Visualizer Diagram" title="Figure 1: Physics Visualizer" >}}

### Renderer
More details can be found [here]({{<ref "/posts/projects/evol/modules/renderer.md">}})

### Game Scenes
For the game scenes, we decided to use the ECS architecture and thus used
[flecs](https://github.com/sandermertens/flecs) due to how fast it is and how
intuitive its API is. This allowed making some per-frame operations that work on
most entities be executed as systems and getting a bit better performance. Also,
having systems reduced the amount of coded needed to specify which entities the
system is supposed to operate on. The most notable systems are:
- Hierarchical transform constraint
- Submission of render-able objects to the renderer

For more details about the game scene and the scene format, look [here]({{<ref "/posts/projects/evol/modules/game.md">}})

### Asset Manager
The asset manager's main responsibility is to provide a unified way to load any
type of asset that we support. The module was designed with extensibility in
mind so that supporting a new asset format is as simple as creating a loader for
it and adding it to the manager's files. Asset loaders are already written for:
- Meshes
- Images
- GLSL Shaders
- Lua Scripts
- JSON files
- Text Files

It also exposes a file-watch functionality that is used to allow hot-loading of
scenes without needing to re-run the entire program whenever an asset changes.

More details about how the asset manager operates internally can be found [here]({{<ref "/posts/projects/evol/modules/asset-manager.md">}})
