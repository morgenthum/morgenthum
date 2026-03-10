+++
title = "Welcome to morgenthum.dev"
date = 2026-03-10
description = "Kicking off my developer blog. I'll be writing about Wild Spikes, Rust, Bevy, and everything I learn along the way."
[taxonomies]
tags = ["meta", "rust", "bevy"]
+++

It's finally here - my little developer blog is online!

## What to expect

I'm currently working on **Wild Spikes**, a 3D game that I'm building with [Rust](https://www.rust-lang.org/) and the [Bevy Engine](https://bevyengine.org/). I constantly run into interesting challenges and new insights that I'd like to share here.

The topics will mostly revolve around **game development with Rust**, but other projects and general software development may show up as well.

## Why Rust and Bevy?

Rust offers a unique combination of performance and safety that is especially valuable in game development. Bevy as an ECS-based engine feels very natural and makes development incredibly productive.

A small example - here's what a simple Bevy system looks like:

```rust
fn move_player(
    time: Res<Time>,
    input: Res<ButtonInput<KeyCode>>,
    mut query: Query<&mut Transform, With<Player>>,
) {
    let mut transform = query.single_mut();
    let speed = 5.0 * time.delta_secs();

    if input.pressed(KeyCode::KeyW) {
        transform.translation.z -= speed;
    }
    if input.pressed(KeyCode::KeyS) {
        transform.translation.z += speed;
    }
}
```

{% callout(kind="tip", title="Stay tuned") %}
In upcoming posts I'll dive deeper into the architecture of Wild Spikes and show how I solve specific gamedev problems with Rust and Bevy.
{% end %}

## The tech stack of this blog

This blog itself is also a small Rust project - it's generated with [Zola](https://www.getzola.org/), a blazingly fast static site generator written in Rust. I simply write posts in Markdown, and Zola takes care of the rest.

See you soon!
