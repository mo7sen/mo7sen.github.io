---
title: "evol Architecture"
description: "The process of picking an architecture for the evol game engine."
date: 2021-07-31T13:25:00Z

categories: ["Project"]
tags: ["Game Engine"]

toc: true

hiddenFromHomePage: true

draft: false
---

## Overview
Any software that is being designed needs to adhere to a specific architecture.
Having an explicit architecture allows the design process to be much smoother as
architectures come with their own set of rules that can increase consistency
across the entire software. A game engine is no different; in fact, as the game
engine is a much bigger software than most, it sometimes needs to have multiple
architectures with some subsystems having their own ones.

## Option #1: Layered Architecture

This architecture is widely used in a lot of software projects. It's also used
in Godot which is currently one of the major game engines. This is mostly
because it focuses on its purpose and does it well, and that is to ensure
maximum stability by having a defined hierarchy for the responsibilities of 
each collection of subsystems in a system. Its main idea is that different 
subsystems can exist on different levels of the software. Having multiple 
levels with subsystems residing on specific ones starts to become useful when 
rules are added to these layers, the most important rule is that subsystems can 
only use functionalities from subsystems that are on a lower layer than they 
are. This allows the dependencies to be a bit more organized as a subsystem 
cannot be dependent on another that is on a higher layer.

{{< figure src="/evol/images/architecture/layered_arch.png" alt="Layered Architecture Diagram" title="Figure 1: Layered Architecture" >}}

However, the main problem with such an approach is that it is static. Static 
here means that once a module changes, all other modules that depend on it are 
affected and might need individual changes to adapt with that change.

## Option #2: Plugin Architecture

This architecture is mainly used by open-source game engines, such as:
WishEngine and MAGE. The plugin architecture’s main idea is that all subsystems 
are considered plugins. These plugins are then connected to a core; which is 
responsible for orchestrating the entire plugin framework and maintaining the 
plugins so that dependencies are always met. This architecture improves 
flexibility and extensibility.

The plugin architecture is on the other end of the spectrum with respect to the 
layered architecture. While the layered architecture has some strict rules 
regarding how subsystems communicate together, the plugin architecture has no 
such rules, and all subsystems can communicate with each other freely. Also, 
while the layered architecture is static, the plugin architecture is dynamic 
enough for subsystems to be added or removed at runtime.

While this architecture beats the layered architecture in some aspects, and 
even brings its own advantages, there are some pros to the layered architecture 
that it just cannot replace. Removing communication criteria ends up 
convoluting the dependencies between systems to the point that it is no longer 
possible to predict what a change to the system will break.

{{< figure src="/evol/images/architecture/plugin_arch.png" alt="Plugin Architecture Diagram" title="Figure 2: Plugin Architecture" >}}

## Option #3: Hybrid Architecture

Since both architectures have both their pros and cons, it is only logical to 
think about building a new architecture that has the pros of both architectures 
and none of their cons. This architecture will be a mix of both with just the 
right amount of each one so that the engine can be both flexible and scale-able.

This architecture can be achieved by first designing a normal plugin framework, 
then enforcing the layered architecture’s strict communication rules through a 
dependency system. Using this design, the engine should have the scale-ability 
of the plugin architecture while also maintaining the maintainability of the 
layered architecture.

{{< figure src="/evol/images/architecture/hybrid_arch.png" alt="Hybrid Architecture Diagram" title="Figure 3: Hybrid Architecture" >}}

## Winning Architecture?

While both the layered architecture and the plugin architecture are very stable 
and tested in widely used game engines, mixing the two architectures together 
gives an architecture with the pros of both and none of their cons. This gives 
us a very stable and scale-able architecture that can be easily maintained.

As shown in **Figure 3**, for such an architecture, there needs to be a single 
point where everything is coupled. This point will be the core of the entire 
engine and can be referred to as the "Plugin Framework" as it will be the part 
of the system that manages the plugins 
(which will be referred to as "Modules".)

# References
- Introduction to Godot development: https://docs.godotengine.org/en/stable/development/cpp/introduction_to_godot_development.html
