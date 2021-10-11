---
title: "evol-renderer"
description: "The evol game engine's renderer module."
date: 2021-07-31T13:25:00Z

categories: ["Project"]
tags: ["Game Engine", "Graphics"]

hiddenFromHomePage: true

toc: true

draft: false
---

### Introduction
The rendering module is one of the more special modules; this is mainly because dependencies were kept to a minimum and extra effort was put to ensure that most of the module was actually built from scratch.

### Implementation
For the graphics API, we used Vulkan. This decision was quite easy to take actually as DirectX wasn’t natively supported in linux and OpenGL wouldn’t provide any experience when it comes to newer graphics APIs which are way more verbose and provide more control over almost everything. 

After playing around with Vulkan for a couple of months, we started to have more confidence in our understanding of the API and decided to start the implementation phase of the renderer. First, we created a thin abstraction layer of most of the Vulkan operations that we needed, and then we decided to look into how the renderer will communicate with the rest of the engine so that it knows what to render. Due to how the data was structured in the ECS module, sending a list of `RenderComponent`s and `TransformComponent`s was almost trivial. And thus was decided to be the main way to register objects for rendering in the next frame.

That’s when the time came for us to settle on what we wanted to have in our `RenderingComponent` structure. The first idea was to simply store all per-object rendering data inside that component and upload-draw-free each frame, it’s quite easy to see why that is a bad idea: GPU bandwidth from repeated uploads and frees of data, more memory usage to store the components, and way too many cache misses when iterating over the said components. The second idea was to simply upload the assets to the GPU memory and simply store handles to those assets in the `RenderComponent` (to be honest, this was the first idea but I just added the super naive one so that I can point out what is wrong with it.) 

While the second option is way better than the first one (and is actually the option used in a lot of game engines until recently), it has one major flaw: too many API calls. Despite API calls being pretty cheap (especially on newer graphics APIs), having a lot of them can introduce some serious overhead. This is mitigated by using a “bindless” architecture, which minimizes the bindings needed per object so that only one binding is needed per object. The gist of this architecture is that instead of binding an asset, assets of the same type (buffers, images) are stored in an array on the GPU. The only binding needed then is for a structure that contains the indices needed by the current object. Those indices can then be used in the shaders to index the assets arrays and get the data easily. 

{{< admonition type=info >}} 
The next steps are DrawIndirect and MultiDrawIndirect which further reduce the bindings and draw calls needed by allowing the GPU to get the data needed for a draw call from a GPU buffer, but we didn’t add those yet so I won’t be talking about them here. 
{{< /admonition >}}

While it was planned to have the rendering pipelines be generated from configuration files, and thus have a pretty flexible way to extend or create pipelines other than the ones provided in the engine by default. This has the added benefit of enabling us to try new rendering techniques without needing to refactor anything in the engine’s code. However, since this project had a deadline, there wasn’t enough time to get to that point and thus we settled on only keeping the built-in pipelines for now. While there are no custom pipelines, custom shaders that are compatible with the default pipelines can be provided by the user and easily compiled at runtime using [libshaderc](https://github.com/google/shaderc).

Then, comes the shading. We started by having a simple forward shading approach. This allowed us to test most of the rendering logic in the most vanilla shading architecture. We decided to use the metallic-roughness PBR model and thus integrated it into the shaders and the materials. Then, after making sure that we are producing correct output, we decided to switch to deferred rendering for the sake of reducing the amount of overdraw that we were doing. Since we had the ability to define multiple built-in pipelines, we managed to keep both the forward shading and deferred shading supported at the same time (I should probably try benchmarking those two approaches so that we can have a better idea of how much performance gain we actually get.)

As for shadows, we decided to use shadow maps due to how simple they are to implement (which was crucial as we were nearing our deadline). We are planning to eventually check out how to integrate shadow volumes in the already existing structure, and then try researching more efficient ways to generate shadows as both of these approaches have their problems.

For antialiasing, we simply decided to settle on FXAA due to how simple it is to implement (a single full-screen pass) and we’re planning to explore more advanced approaches, like temporal anti-aliasing, at some point in the future.

## Things to add in the future

Some of the things that are currently being considered to be added in the future:
- DrawIndirect/MultiDrawIndirect support
- Configurable Pipelines
- Ability to update assets without needing to upload all the assets all over again (This can probably be achieved through generational indexing and a way to get the next free cell in the asset array, for example a list of free cells)
- Custom behaviour for objects that are not expected to change/move (static objects)
- UI rendering (Will probably be done using imgui because of how modular it is and the possibility to generate Lua bindings for it and start playing with it in game scripts)
- Replace the deferred shading’s G-Buffer with a visibility buffer
- Add an alpha pass to allow the rendering of translucent objects
