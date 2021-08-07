---
title: "evol-assetmanager"
description: "The evol game engine's asset-manager module."
date: 2021-07-31T13:25:00Z

categories: ["Project"]
tags: ["Game Engine"]

toc: true

hiddenFromHomePage: true

draft: false
---

The asset managers try to provide some features that make the workflow of
adding/using assets less painful. While there is no fancy features like
asynchronous loading or callbacks, the base foundation of the asset manager is
quite flexible and allows the addition of such features in the future.

## Mounts
Ok, so it's probably not the best idea to start this off with a QoL feature, but
it's too fun to leave for later. The feature is basically virtual mounting of
folders (or ZIP files) to a specific mount point so that items in that folder
can be referenced relative to the mount point. For example, a user can specify
the directory at which the shaders reside in the *game.proj* file like this:
```json
"mounts": [
  {
    "path": "./assets/shaders",
    "mountpoint": "shaders"
  }
]
```
This way, the default fragment shader can then be referenced as 
`shaders://default.frag`, which would be pretty useful when dealing with long
paths. We got inspired to add this feature when [ÖbEngine](https://github.com/ObEngine/ObEngine) 
added it and thought that it'd be a nice-to-have.

## Filewatch
Another feature was the ability to monitor mounted directories. This allowed the
dispatching of events whenever an asset is changed which, in turn, allowed the
automatic reloading of said assets. This was used to enable hotloading for
scenes, shaders and scripts.

## Storage
Assets are stored in an ECS world, which might seem counter-intuitive but was
actually found to be quite efficient when operations need to be made on all the
assets, like the lifetime counter. While all assets are reference-counted,
assets that are used frequently by a single entity will just keep on being
loaded and unloaded each frame. To avoid that, a lifetime counter was added that
gets decremented each frame whenever the reference-count is 0. By setting the
lifetime of assets to 2 by default, assets can now survive for 2 frames after
their reference-count reaches zero, then they're freed.

## Asset Loading
The assets are loaded first as generic binary files and handles are returned.
After that, those handles can be passed to a loader so that it can return an
object that interprets that binary as a specific asset type. The currently
implemented loaders are:
- **Mesh Loader**: Interprets the binary as our own mesh format and extracts all
    the data from it
- **Image Loader**: Interprets the binary as our own image format and extracts
    all the data from it
- **Shader Loader**: Interprets the binary as a GLSL shader and compiles it to
    SPIR-V using shaderc.
- **JSON Loader**: Interprets the binary as a JSON file, parses it, and returns
    our very own evjson context.
- **Text Loader**: Interprets the binary as a simple text file and puts it in a
    dynamic string (evstring)
