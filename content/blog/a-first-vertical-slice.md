+++
title = "A first vertical slice"
date = 2026-09-28
description = "So my last update on the game was already a while ago. Here is a video of how it looks at the moment."
[taxonomies]
tags = ["rust", "bevy", "wild-spikes"]
+++

So my last update on the game was already a while ago. Here is a video of how it looks at the moment.

<video controls playsinline preload="metadata" width="1662" height="1080" poster="/videos/wild-spikes_2026-09-28.jpg" aria-label="Wild Spikes: A first vertical slice gameplay video">
  <source src="/videos/wild-spikes_2026-09-28.mp4" type="video/mp4">
  <a href="/videos/wild-spikes_2026-09-28.mp4">Download the gameplay video (MP4).</a>
</video>

I implemented a first small chapter as a vertical slice and played around with lighting, LODs, 3D models and sounds, as you can see and hear.

The biggest boost for development was using BSN files to make progress in a "proper" data-driven way. Based on this, I could build the chapter, the scene, and also the story and NPC behavior, using my existing engine crates (e.g. lod, weather, etc.). I also switched the audio to [bevy_seedling](https://github.com/corvusprudens/bevy_seedling) and implemented different effects to make everything feel more immersive.

Here and there I still have performance problems, but often this is because of things like too many high-resolution 3D models (even at low LOD). Another problem was that I was doing mesh picking on all objects because I ignored the markers. So, lots of stupid problems.
