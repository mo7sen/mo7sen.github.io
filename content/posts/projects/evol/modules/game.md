---
title: "evol-game"
description: "The evol game engine's game module."
date: 2021-07-31T13:25:00Z

categories: ["Project"]
tags: ["Game Engine"]

hiddenFromHomePage: true

toc: true

draft: false
---

The game module is where most of the fun actually happens; the game module 
attempts to use the other modules to make the program behave as a game engine. 
Through the combination of the world module, the input module, the physics 
module, the script module, the asset module and the rendering module, the game 
module manages to connect all these modules together to make an actual game that 
runs.

First of all is the scene representation; in the game module, a scene is an 
isolated component that cannot interact with other scenes. To fulfill that 
isolation, instances from data needed to be used so that different scenes don’t 
share the same storage. That’s why the Scene type ended up being mainly a 
combination of a physics world, an ECS world, and a scripting context. A scene 
consists of more data that it might need when running, like the ID of the active 
camera, but those are not something that needs to be talked about as they are 
implementation details rather than design choices.

The scenes are stored in a map with their names as identifiers. This allows for 
easy referencing of scenes through their names. This map is then cleared with 
all the scenes destroyed at the end of execution when the game module is 
unloaded from memory. The active scene is also stored in the global data of the 
game module since it’ll mostly be the one to be used for any operations and thus 
having it cached will reduce any overhead from hashing the scene name for each 
operation. Each frame, the active scene is updated and, in turn, updates all of 
its inner structures (the ECS world, the physics world, and the scripting 
context). Whenever a new scene is created, it initializes its internal 
structures so that it’s ready to progress when needed. Also, it checks to see 
whether there is an already active scene or not; if there is no active scene, 
then the newly created scene is set to be active.

The game module also provides the base definition for a game object; whenever a 
new object is created, a name and a transform component are attached to it as 
that is the bare minimum requirement for an object to be considered a game 
object (to be named and to have a transform). Having a name allows the objects 
to have an identifier that they can be easily referenced with whenever that 
specific object is needed.

The transform component was a bit tricky to come up with. We needed a way to 
have the transform be simple to change in a readable manner for scripting (using 
position, Euler angles, and scale) but we also didn't want to need to 
recalculate the world-space transform matrix whenever we need it by walking up 
the scene hierarchy until we reach the root. That’s why we decided to use a 
dirty flag for the transform. The idea is simple, the transform component 
contains the local position, local rotation, local scale, and world-space 
transform matrix of the entity. However, it also contains a dirty flag that is 
set whenever any of the local transform parameters are changed, signifying that 
the world-space transform matrix needs to be recalculated. 

This means that a simple `setPosition` operation goes as follows:
- *Set the local position of the entity*
- *Set dirty bit for entity (and recursively to all its children)*

The dirty bit is set for children as their world-space transform will change 
when that of the parent is. Notice that the world-space matrix is not updated 
yet; it will only get updated when a module asks for it. This helps make 
multiple local transform changes more efficient as it makes subsequent changes 
to the local transform not affect the world transform until needed. A sample 
`getWorldTransform` is as follows:
- *Get transform component*
- *Check if dirty bit is set*
- *If bit is set, then update the world matrix (and in turn any dirty parents)*
- *Return the world matrix*

The way the world transform of an object is updated is by checking the dirty 
flag first. If the dirty flag is not set, then the transform is already up to 
date. However, if the dirty bit is set, then a local transform matrix is 
calculated and is transformed with the parent’s world transform matrix. If the 
parent’s dirty bit flag is set, then it is updated and this operation keeps on 
moving up the hierarchy until a non-dirty transform is found and is used to 
update the children down the hierarchy until the wanted transform is up to date.

Also, a camera type is provided by the game module to allow the ease the use of 
cameras by providing more utility functions. A camera component is available so 
that cameras can be dealt with as normal entities with an extra component. The 
camera component contains the projection matrix, the view matrix, and some 
tune-able camera parameters that can be set through utility functions. An `OnSet` 
listener is added on the camera component so that whenever a tune-able parameter 
is changed, the projection matrix is recalculated to stay up to date. As for the 
calculation of the view matrix, it is simply calculated by getting the inverse 
of the world-space transform matrix of the camera. Whenever a new camera is 
created, the scene is checked to see whether there is an active camera; if 
there’s no active camera, then the newly created one is set to be active.

For the scripting API, a lot of functions are exposed to ease scripting and make it a lot easier for scripts to do simple operations. Some of the exposed functions are:
| Function                     | Description                                                                                                                                                                  |
|------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ev_object_getname`          | Used with Lua’s meta-tables to allow the reading of an entity’s name by doing `this.name`                                                                                     |
| `ev_object_getposition`      | Used with Lua’s meta-tables to allow the reading of an entity’s position using `this.position` which returns the exposed `Vec3` type                                          |
| `ev_object_setposition`      | Used with Lua’s meta-tables to allow the writing of an entity’s position using `this.position` which takes a `Vec3`                                                           |
| `ev_object_getworldposition` | Used with Lua’s meta-tables to allow the reading of an entity’s global position using `this.worldPosition` which returns a `Vec3`                                             |
| `ev_object_seteuler`         | Used with Lua’s meta-tables to allow the writing of an entity’s rotation using `this.eulerAngles` which takes an Euler angle represented by a `Vec3`                          |
| `ev_object_getforwardvec`    | Used with Lua’s meta-tables to allow the reading of an entity’s global forward direction using this.forward which returns a `Vec3`                                            |
| `ev_object_getrightvec`      | Used with Lua’s meta-tables to allow the reading of an entity’s global right direction using this.right which returns a `Vec3`                                                |
| `ev_object_getupvec`         | Used with Lua’s meta-tables to allow the reading of an entity’s global upward direction using this.up which returns a `Vec3`                                                  |
| `ev_game_setactivescene`     | Wrapped in a Lua function called `gotoScene(sceneName)` which takes a scene name and switches to that scene                                                                  |
| `ev_scene_getobject`         | Wrapped in a Lua function called `getObject(objectPath)` which takes the path of an object (including its parents’ names) and returns that object                            |
| `ev_sceneloader_loadprefab`  | Wrapped in a Lua function called `loadPrefab(filePath)` which takes the path of a prefab in the file-system, loads into the scene and returns that the object to the script   |


Now that the scene is pretty much operational, the thing that was left was to 
allow the scenes to be loaded from scene files that are more readable than plain 
code and don’t require recompilation for every single change made to the scene. 
The scene format that we came up with was JSON-based to ensure high readability 
and ease of changing scene parameters. A typical scene looks like the following:
```json
{
  "id":"MainScene",
  "nodes": [
    ...
  ],
  "materials": [
    ...
  ],
  "pipelines": [
    ...
  ],
  "activeCamera": "Camera"
}
```


In the scene file shown above, both the “materials” and “pipelines” list are 
irrelevant for the game module. These lists are passed as-is to the rendering 
module so that it can parse them as it sees fit. The fact that the game module 
doesn't care about what’s in those lists makes it easier for the rendering 
module to update what it expects from that JSON without the game module needing 
to mirror those changes.

As for the nodes, they’re a structure that contains a node’s name, its 
components and its children. A typical node looks like the following:
```json
{
  "id": "Box",
  "components": [
    ...
  ],
  "children": [
    ...
  ]
}
```


The children list is basically other nodes that are defined the same way 
recursively until leaf nodes are found that have no children and, in turn, have 
no children list. As for the components list, there are a few components that 
are exposed to the scene file, those components are:

- Transform Component
```json
{
  "type": "TransformComponent",
  "position": [0.0, 0.0, 0.0],
  "rotation": [0.0, 90.0, 0.0],
  "scale": [1.0, 1.0, 1.0]
}
```


- Script Component
```json
{
  "type": "ScriptComponent",
  "script_name": "BoxScript",
  "script_path": "scripts://MainScene/box.lua"
}
```


- Camera Component
```json
{
  "type": "CameraComponent",
  "view": "Perspective",
  "fov": 90,
  "near": 0.001,
  "far": 1000
}
```


- Rigidbody Component
```json
{
  "type": "RigidbodyComponent",
  "rigidbodyType": "Dynamic",
  "mass": 1.0,
  "restitution": 1.0,
  "collisionShape": {
    "type": "Box",
    "halfExtents": [1.0, 1.0, 1.0]
  }
}
```


- Render Component
```json
{
  "type": "RenderComponent",
  "material": "WhiteMaterial",
  "mesh": "assets://meshes/box.mesh"
}
```


- Light Component
```json
{
  "type": "LightComponent",
  "color": [1.0, 1.0, 1.0, 1.0],
  "intensity": 10
}
```

Prefab files are also JSON files. Prefabs have the same structure as normal 
nodes with the sole difference that prefabs are stored in their own files while 
nodes need to be stored inside a scene. Prefabs can be loaded using the same 
procedures that the scene uses when loading the nodes.
