---
title: "RusTracer"
date: 2021-07-31T13:25:24Z

description: "My first raytracer"

categories: ["Project"]
tags: ["Graphics"]
featuredImagePreview: "rustracer/preview.png"

draft: false
---

In an effort to learn about ray tracing, I decided to follow ssloy’s tinyraytracer tutorial which is, in my opinion, a very gentle introduction into the world of raytracing. Despite the tutorial using C++, I decided to use Rust for no reason except that I hadn’t written any Rust code in a while and felt a bit rusty (pun intended). 

Another thing that I did differently from the tutorial is that I rendered to an SDL window instead of rendering to an image. This gave me the ability to have the camera be dynamic and move around freely in the scene.

Within a few days, I had already finished the tutorial and was ready to try adding more stuff. I started by adding some multithreading due to how trivially parallelizable a ray-tracer is. This noticeably decreased the frame time enough for the framerate to reach interactive rates at a low enough resolution (I unfortunately don’t have any benchmarks as I forgot to make them, but since the raytracer is already working it should be pretty easy to benchmark anything if the need arises).

I also added a couple of extra primitives:
- Disks (which were pretty trivial as I already had planes)
- Axis-aligned boxes (that could be used later for the BVH)

After that I planned to put a lot of extra features:
- Multisample Anti-aliasing
- Transformations
- UV mapping
- Triangle Meshes
- Denoising
- Acceleration Structure (BVH)
- Try pushing most of the work to the GPU (planned to use Vulkan compute shaders.)

However, since this was a beginner tutorial and this was my first time making a raytracer, I ended up with a program architecture that wouldn’t allow the addition of most of this stuff. So since I decided that this raytracer wouldn’t be my last, I kept these features for the next one.

Some Screenshots:
{{< figure src="/rustracer/preview.png" alt="Thumbnail Image" title="Figure 1: Thumbnail Image">}}
{{< figure src="/rustracer/disks-and-aabb.png" alt="Screenshot showing disks and axis-aligned boxes" title="Figure 2: Disks and Axis-aligned boxes" >}}
{{< figure src="/rustracer/noise.png" alt="Screenshot showing noise with too many reflections" title="Figure 3: Noise" >}}

The source code for this raytracer can be found in [this
repository](https://github.com/mo7sen/rustracer).
