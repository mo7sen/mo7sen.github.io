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

## Implementation

### Core
More details can be found [here]({{<ref "/projects/evol/modules/core.md">}})
### Scripting
More details can be found [here]({{<ref "/projects/evol/modules/scripting.md">}})
### Physics
More details can be found [here]({{<ref "/projects/evol/modules/physics.md">}})
### Renderer
More details can be found [here]({{<ref "/projects/evol/modules/renderer.md">}})
### World
More details can be found [here]({{<ref "/projects/evol/modules/world.md">}})
### Asset Manager
More details can be found [here]({{<ref "/projects/evol/modules/asset-manager.md">}})
