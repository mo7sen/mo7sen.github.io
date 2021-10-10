---
title: "RustyNinja"
date: 2021-07-31T13:25:24Z

description: "My entry for RDJ2019"

categories: ["Project"]
tags: ["Game", "Unity"]

featuredImagePreview: "rustyninja/preview.png"

draft: false
---

It started with me finding out that RDJ2019 (Game showcase event) was around the corner and deciding that I wanted to submit an entry for a game that I’d like to play. So I started thinking about a game idea until I remembered [this clip](https://www.youtube.com/watch?v=AKiDgJxFRs8). I liked the idea of a ninja jumping on missiles while simultaneously destroying them, and that’s what I decided to make.

So the idea was as follows: you’re a ninja in a peaceful village, an army is attacking the village to get rid of its residents and control its resources, you start attacking the missiles that are shot at the village to make sure that none of them reach their target. This is an endless game with the score being the time survived by both the ninja and the village. I decided to make it 2D because it felt more manageable, especially because I had less than two weeks to build the game.

The reason this game is called Rusty Ninja is mainly because of how I decided to make it in Rust. Having discovered the language just recently and barely doing anything with it, I thought it would be a good idea to give making a game with it a try. However, since the libraries that existed at that time were too low level for me, I got a bit overwhelmed. After a week of playing around with Rust (and honestly learning a lot), all I could come up with was: 
{{< figure src="/rustyninja/rust_ver_1.gif" alt="Progress in Rust GIF 1" title="Figure 1: Progress in Rust 1" >}}
{{< figure src="/rustyninja/rust_ver_2.gif" alt="Progress in Rust GIF 2" title="Figure 2: Progress in Rust 2" >}}

While that might not seem too bad, with the tight deadline that I had for the submission, I wouldn’t be able to make it in time. So I switched to Unity.

That’s when I started to appreciate the speed of Unity when it comes to the speed of prototyping. It was pretty obvious, especially after seeing the progress I made in about an hour.
{{< figure src="/rustyninja/playing_around_with_mechanics.gif" alt="Playing Around GIF 1" title="Figure 3: Playing Around 1" >}}
{{< figure src="/rustyninja/playing_around_with_mechanics2.gif" alt="Playing Around GIF 2" title="Figure 4: Playing Around 2" >}}

As can be seen from the previous GIFs, the player would have the ability to:
- Run
- Jump
- Throw shurikens
- Throw rope shuriken

While using a ball as a stub for the player, I decided to try using rigid body physics for the movement of the ball. As soon as I unleashed the power of physics, I noticed that this game could be made way harder by removing the ability to jump. This was possible because of how the weight of the player affected the missiles: if the player managed to tilt a missile so that it can be used as a ramp, moving fast enough could propel them at the angle of the tilt. This was also when it became clear that having the player be a ball would be better than having a humanoid.

Some more rules were added to the game at that point, for example:
- If a missile passes the screen, that means the village was destroyed
- If the ninja falls in the water, that’s it for them
- Hitting a missile with a shuriken on the head causes it to explode
- Hitting a missile with a shuriken in the thruster causes it to stop moving forward and gradually fall into the water
- If a missile’s thruster flame touches the head of another missile, the latter shall explode
- If a missile’s head touches another missile’s head, they both explode

Several stages of the development of the game are shown below using images and GIFs:
- Funny bugs
  {{< figure src="/rustyninja/rope_bug.gif" alt="Rope Bug" title="Figure 5: Rope Bug" >}}
- Explosion Animation
  {{< figure src="/rustyninja/explosion_animation.png" alt="Tons of explosions" title="Figure 6: Explosion animation" >}}
- Minimal Level Art
  {{< figure src="/rustyninja/min_level_art.png" alt="Minimal Level Art" title="Figure 7: Level Art" >}}
- Twin Ninja Mode (Luckily thrown away)
  {{< figure src="/rustyninja/twin_ninja_mode.gif" alt="Gameplay GIF of twin-ninja mode" title="Figure 8: Gameplay of twin-ninja mode" >}}

And eventually, I ended up with the game shown below.
{{< youtube YJLQH4KXrX4 >}}

While the game wasn’t accepted for RDJ2019, I sent this game to multiple friends during its development and they all got addicted to it for at least a couple of days, which I consider a complete win.

Here, take an art piece.
{{< figure src="/rustyninja/artpiece.png" alt="Early game wallpaper" title="Figure 9: Rusty Ninja Wallpaper" >}}

