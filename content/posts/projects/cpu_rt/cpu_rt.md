---
title: "CPU RayTracer"
date: 2021-07-31T13:25:24Z

description: "My second raytracer"

categories: ["Project"]
tags: ["Graphics"]
featuredImagePreview: "cpu_rt/preview.png"

draft: false
---

## Introduction
After finishing my [first raytracer](https://mo7sen.github.io/projects/rustracer), it was obvious that I needed to make another one if I wanted to learn more about the topic. However, since I wasn’t confident enough to step out of the tutorial territory, I decided that I would use [Peter Shirley’s RTOW series](https://raytracing.github.io). 

One of the major advantages of this series is that while it is introducing the concepts in a gentle manner, it is also giving advice on how to structure the project so that adding more features in the future doesn’t end up being a huge task with tons of refactors. Combined with the fact that this is my second raytracer, this meant that I could have a much nicer code structure for this one.

## Implementation
After going through the first two books, I had a much more efficient raytracer
than my first one with more features. Some of these features included:
- Antialiasing
- Motion blur
- Volumetric fog
- Emissive materials
- Bounding volume hierarchy

### Some screenshots
{{< figure src="/cpu_rt/screenshot_base_4.png" alt="Screenshot 4" >}}
{{< figure src="/cpu_rt/screenshot_base_3.png" alt="Screenshot 3" >}}
{{< figure src="/cpu_rt/screenshot_base_2.png" alt="Screenshot 2" >}}
{{< figure src="/cpu_rt/screenshot_base_1.png" alt="Screenshot 1" >}}

{{< admonition type=note >}}
While I was planning to go through the whole series, a quick skim through the
third book made it clear that it's way more advanced than the first two. While
I'm still planning to go through it, I'm postponing it until I either need to
read it or until I have more time to spare.
{{< /admonition >}}

## Additional stuff I added
After finishing the first two books, the time came for me to start playing
around and adding more features to the raytracer.

### Custom Render Targets
While the books expected the output to be exported to a PPM image file, I
abstracted it into a generic `RenderTarget` instead. This had two advantages:
1. If I ever decide to make the raytracer more interactive, I'll need to start
   outputting frames to a window. To avoid major refactors, the window can
   simply be a `RenderTarget` and be quite trivial to replace the image output
   with it.
2. For most of the development time, I was developing the raytracer on my own
   headless server and hosting the output image through the browser. This meant
   that I needed a more common format (e.g. PNG) so having the `RenderTarget`
   be changeable on demand was quite useful.

### Multi-threading
Due to how rays are their own independent entities that traverse the scene
without caring for each other, splitting the rays over multiple threads is quite
trivial. Since there were no concurrency problems that could occur, I settled on
using OpenMP as it was easy to use and efficient when the parallelization is 
trivial enough. I ended up splitting the work on the scanline level; this meant
that each thread would be responsible for a set of rows in the image.

### Triangle Meshes
Having seen enough spheres throughout the development, I decided that it was
time for me to display more complex stuff. Having triangle meshes required me to
first find a way to test the intersection with an individual triangle. For this
I used scratchapixel's nice
[guide](https://www.scratchapixel.com/lessons/3d-basic-rendering/ray-tracing-rendering-a-triangle/ray-triangle-intersection-geometric-solution)
on how to implement ray-triangle intersections.

{{< figure src="/cpu_rt/ray_triangle_intersection.png" alt="Ray-Triangle Intersection" title="Figure 1: Ray-Triangle Intersection">}}

### OBJ Loading
After that was finished, I needed to start loading meshes. For that, I decided
to start with the OBJ format then my work my way up. For the OBJ, I got
[tinyobjloader](https://github.com/tinyobjloader/tinyobjloader) and started
loading the meshes and materials from it.

{{< figure src="/cpu_rt/obj_1.png" alt="OBJ Loading: African Head" >}}
{{< figure src="/cpu_rt/obj_2.png" alt="OBJ Loading: Rock" >}}
{{< figure src="/cpu_rt/obj_3.png" alt="OBJ Loading: Stone Golem" title="Figure 2: Loading OBJ Models">}}

### GLTF2 Loading
I knew that if I wanted to have more complex scenes being rendered, I'd need a
more complex format like GLTF2. So I got
[tinygltf](https://github.com/syoyo/tinygltf) and started loading GLTF files.

{{< figure src="/cpu_rt/gltf_1.png" alt="GLTF Loading: Damaged Helmet (No shading)" >}}
{{< figure src="/cpu_rt/gltf_2.png" alt="GLTF Loading: Textured Box (UV Mapping)" >}}

{{< admonition type=note >}}
One of the major problems I had was the low performance that a high-poly model
brought with it. This was easily fixed by changing the way primitives were
stored inside a mesh. Instead of having a mesh be a list of mesh primitives, I
changed it to be a BVH of primitives. This, combined with some pre-processing in
blender to increase the number of primitives, led to a huge jump in performance
when rendering the damaged helmet GLTF model.
{{< /admonition >}}

### PBR Materials
One of the reasons that I switched from OBJ models to GLTF was how limited the
material model was for OBJ. With GLTF, I had more flexibility due to it
supported the metallic-roughness pipeline by default (and having extensions for
other pipelines). So I started making a PBR material; this involved:
- Diffuse mapping
- Normal Mapping
- Emissive Mapping
- Metallic-Roughness

Unfortunately, the scene format in this raytracer wasn't ready for a PBR
pipeline as I had no way to know whether I'm hitting a light or not, so I
settled on having an approximation that, while inaccurate, doesn't look too bad.

{{< figure src="/cpu_rt/gltf_3.png" alt="GLTF Loading: Damaged Helmet (PBR)" >}}

## Final Render
{{< figure src="/cpu_rt/preview.png" alt="Final Render (Thumbnail Image)" >}}

## Things to add in the future
- Change the structure of the BVH so that it is a contigious array so traversal
  is more cache-friendly
- GPU Acceleration (Probably, Vulkan Compute shaders)
- True PBR
- Concepts from the third book


The source code for this raytracer can be found in [this repository](https://github.com/mo7sen/cpu-raytracer).
