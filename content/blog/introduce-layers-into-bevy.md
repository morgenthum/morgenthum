+++
title = "How to introduce layers into Bevy games"
date = 2026-03-10
description = "A look at the layered architecture behind Wild Spikes, my survival game built with Rust and Bevy. Strict dependency rules, clear boundaries, and why it matters."
[taxonomies]
tags = ["rust", "bevy", "wild-spikes", "architecture"]
+++

Wild Spikes is a survival and stealth game with quests, day/night cycles, weather, crafting, and other systems you would expect in a full-fledged game. I'm building it with Rust and Bevy, targeting PC and mobile. As the project grows, keeping things organized becomes critical. In this post I want to share the architecture I use and the thinking behind it.

## The problem

Bevy makes it easy to add systems and plugins. I started the project as one crate, with a few plugins that all lived together, operating on one State, SubState, or ComputedState. It was great for getting something up and running quickly. As the plugins grew, it became harder to keep track of dependencies. The moment I introduced the HUB - a separate screen for opening the inventory and crafting menu - the whole thing became messy. I couldn't test it because everything was wired together, either through direct dependencies or through shared states. Sometimes my character controller blew up, sometimes the HUB exploded, and every time I had to fix something while keeping the whole picture in mind. I didn't just want Bevy plugins - I wanted real self-contained modules. So I split the codebase into multiple crates. Some were more game-specific, others more generic. After a few iterations, I introduced a separate "engine" layer to abstract some of the game-specific details and keep things inside the game cleaner and more focused. After further iterations, I arrived at the architecture I'm currently using. I'm sure it's not perfect, but it has been working well for me so far.

## The layers

The architecture is organized in layers, ordered from the bottom up. Each layer can only depend on layers below it - never upward.

```
Product Apps        apps/*          Platform entries (desktop, mobile)
Sandboxes           sandboxes/*     Prototyping, experiments
Game Assembly       assembly/       Composition root, states, scheduling
App Bootstrap       bootstrap/      Bevy defaults, window, logging
Game Features       game/*          Survival, quests, NPCs, UI
Game Core           game/core       Shared contracts, events, tags
Engine Modules      engine/*        Generic tech, no game knowledge
```

## What lives where

**Engine Modules** are the foundation. These are reusable building blocks that know nothing about the actual game. Things like a camera controller, input actions, weather/time simulation, navigation, animation utilities, and visual effects. They work with generic components and traits. You will never find a `Snail` or `Fox` type here.

**Game Core** defines the shared contracts. Data structures, traits, events, and marker components that game features need to communicate at the lowest level. It contains no gameplay logic itself - just the vocabulary that features use to talk to each other.

**Game Features** is where the actual game lives. Each feature - character, NPCs, narrative, survival, UI - is its own crate with its own plugin. Features depend on Game Core and Engine Modules, and can depend on each other when explicitly needed.

**App Bootstrap** handles the Bevy boilerplate: default plugins, window setup, logging, settings I/O. Both the real game and sandboxes use this.

**Game Assembly** is the composition root. It wires everything together: which plugins to load, which states exist, how system sets are ordered, and which feature flags are active. No gameplay logic here - just configuration.

**Sandboxes** are throwaway apps for prototyping. They can reach into any layer, which makes them great for quick experiments without polluting the real codebase.

**Product Apps** are the final binaries. A desktop app, a mobile app. They do almost nothing - just connect Bootstrap and Assembly.

## The dependency rules

The core principle is simple: dependencies only point downward. But there are a few extra rules that matter:

- Engine Modules must never reference concrete game types. No `Owl`, no `QuestLog`. Only generic components and traits.
- Game Features must not use closed enums that grow with every new content type. Instead, we use composition via components. Each concrete type builds its own set of behavior components.
- Game Core defines generic contracts. Concrete implementations belong in the features.
- Product Apps contain no gameplay logic. They only wire Bootstrap and Assembly.

These rules are not just documented - they are enforced. A custom `xtask` lint checks the dependency graph on every CI run.

## Communication between features

Features do not reach into each other's internals. Instead, they communicate through clear public APIs:

- **Events and Messages** are the main tool for cross-feature communication. One feature sends a `StartDialogue` or `GiveItem` event, another reacts to it. No direct coupling needed.
- **Public components** allow features to spawn entities that other features understand. A feature can attach a marker component that another feature's systems pick up.

Direct queries into another feature's private components are not allowed. If it is not part of the public API, it does not exist for other features.

## Scheduling

One important aspect is the scheduling of systems. I thought my plugins were independent, but they were not. Even after splitting them into separate crates, they were still tightly coupled through states - which were not available in isolated examples or sandbox apps. So I introduced injectable schedules for each plugin. It looks like this:

```
pub struct CharacterPlugin {
    spawn_schedule: Interned<dyn ScheduleLabel>,
    despawn_schedule: Interned<dyn ScheduleLabel>,
    input_schedule: Interned<dyn ScheduleLabel>,
    interact_schedule: Interned<dyn ScheduleLabel>,
    camera_collision_schedule: Interned<dyn ScheduleLabel>,
}
```

The `CharacterPlugin` combines all character-related systems, such as camera following, collision, hovering, ground adjustment, and so on.

The initialization in the game assembly looks like this:

```
app.add_plugins((
    CharacterPlugin::new(
        OnEnter(AppState::InGame),
        OnExit(AppState::InGame),
        Update,
        Update,
        PostUpdate,
    ),
    NpcPlugin,
    OwlPlugin::new(OnEnter(AppState::InGame)),
));
```

But in examples or sandbox apps it can be initialized like this:

```
app.add_plugins(CharacterPlugin::new(
        OnEnter(SandboxState::Running),
        OnExit(SandboxState::Running),
        Update,
        Update,
        PostUpdate,
    ))
```

That way, the functionality of my self-contained crates is truly self-contained. I can test them in isolation without having to worry about the rest of the game.

## Why it matters (for me)

* Every crate in every layer can have its own examples. This lets me test them in isolation.
* Through isolated testing, I can clearly see when something feels messy or when the API is lacking. If I have to reach into another crate or add a bunch of unrelated plugins, it is a sign that the public API of that crate is not good enough.
* Smaller crates mean faster incremental builds. When I change something in a game feature, only that crate and the layers above it need to recompile - not the entire project.
* Clear boundaries make it easier to reason about changes. I can refactor the internals of a feature without worrying about breaking other features, as long as the public API stays the same.
* The engine modules are not tied to Wild Spikes. I can reuse them in other projects or share them as standalone crates.
* Sandboxes let me prototype new ideas quickly. I can try out a new mechanic or visual effect without touching the real game code - and throw it away if it doesn't work out.