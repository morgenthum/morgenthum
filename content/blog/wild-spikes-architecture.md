+++
title = "How I structure Wild Spikes — a layered architecture for Bevy games"
date = 2026-03-10
description = "A look at the layered architecture behind Wild Spikes, my survival game built with Rust and Bevy. Seven layers, strict dependency rules, and why it matters."
[taxonomies]
tags = ["rust", "bevy", "wild-spikes", "architecture"]
+++

Wild Spikes is a survival and stealth game with quests, day/night cycles, weather, crafting, and an NPC system. I'm building it with Rust and Bevy, targeting PC and mobile. As the project grows, keeping things organized becomes critical. In this post I want to share the architecture I use and the thinking behind it.

## The problem

Bevy makes it easy to add systems and plugins. A bit too easy, maybe. Without rules, you end up with a codebase where everything depends on everything. Engine code imports game types, game features reach into each other's internals, and adding a new NPC type means editing five files across the project.

I wanted clear boundaries. Something where I can work on the weather system without worrying about breaking quests, and where adding a new item type does not require changes in existing enums or match arms.

## Seven layers

The architecture is organized in seven layers, numbered from the bottom up. Each layer can only depend on layers below it — never upward.

```
L7  Product Apps        apps/*          Platform entries (desktop, mobile)
L6  Sandboxes           sandboxes/*     Prototyping, experiments
L5  Game Assembly        assembly/       Composition root, states, scheduling
L4  App Bootstrap       bootstrap/      Bevy defaults, window, logging
L3  Game Features       game/*          Survival, quests, NPCs, UI
L2  Game Core           game/core       Shared contracts, events, tags
L1  Engine Modules      engine/*        Generic tech, no game knowledge
```

Content (assets, definitions, levels) sits outside this hierarchy as data — it has no code.

## What lives where

**Engine Modules** (L1) are the foundation. These are reusable building blocks that know nothing about the actual game. Things like a camera controller, input actions, weather/time simulation, navigation, animation utilities, and visual effects. They work with generic components and traits. You will never find a `Snail` or `Sword` type here.

**Game Core** (L2) defines the shared contracts. Data structures, traits, events, and marker components that game features need to communicate. It contains no gameplay logic itself — just the vocabulary that features use to talk to each other.

**Game Features** (L3) is where the actual game lives. Each feature — character, NPCs, narrative, survival, UI — is its own crate with its own plugin. Features depend on Game Core and Engine Modules, and can depend on each other when explicitly needed.

**App Bootstrap** (L4) handles the Bevy boilerplate: default plugins, window setup, logging, settings I/O. Both the real game and sandboxes use this.

**Game Assembly** (L5) is the composition root. It wires everything together: which plugins to load, which states exist, how system sets are ordered, and which feature flags are active. No gameplay logic here — just configuration.

**Sandboxes** (L6) are throwaway apps for prototyping. They can reach into any layer, which makes them great for quick experiments without polluting the real codebase.

**Product Apps** (L7) are the final binaries. A desktop app, a mobile app. They do almost nothing — just connect Bootstrap and Assembly.

## The dependency rules

The core principle is simple: dependencies only point downward. But there are a few extra rules that matter:

- Engine Modules must never reference concrete game types. No `Owl`, no `QuestLog`. Only generic components and traits.
- Game Features must not use closed enums that grow with every new content type. Instead, we use composition via components — each concrete type builds its own set of behavior components.
- Game Core defines generic contracts. Concrete implementations belong in the features.
- Product Apps contain no gameplay logic. They only wire Bootstrap and Assembly.

These rules are not just documented — they are enforced. A custom `xtask` lint checks the dependency graph on every CI run.

## Communication between features

Features do not reach into each other's internals. Instead, they communicate through:

- **Events** for cross-feature messages (like `StartDialogue` or `GiveItem`)
- **Resources as services** (like a `WeatherService` or `QuestService`)
- **Observers** when it fits

Direct queries into another feature's private components are not allowed.

## Scheduling

Bevy runs systems across multiple schedules. Without rules, it is easy to end up with physics running during a menu or input being processed one frame late. Wild Spikes uses a fixed pipeline:

- **PreUpdate** — Input sampling, camera input, intent calculation
- **FixedUpdate** — Simulation: physics, AI ticks, survival mechanics
- **Update** — Presentation: animations, VFX, UI updates, narrative
- **PostUpdate** — Camera smoothing, debug rendering

Every gameplay system is state-gated. Nothing from `InGame` runs during menus or loading screens. This gating is configured per-schedule in the Assembly.

## Plugin conventions

Every engine module and game feature exposes the same surface:

- A `Plugin` (one per crate)
- A `SystemSet` (for ordering)
- Optionally a config struct and a prelude

Plugins that need lifecycle hooks (spawn/despawn) take a schedule parameter in their constructor. The Assembly decides when things happen — `OnEnter(AppState::InGame)` for the real game, `OnEnter(SandboxState::Running)` for a sandbox. The plugin itself does not hardcode this.

## Data-driven content

Items, quests, NPC definitions, and spawn tables are defined in RON files, not in Rust code. This means adding a new item is a data change, not a code change. Unknown IDs or missing assets produce a warning and get skipped — the game never panics because of content.

## Why this matters

The architecture adds some upfront work. Every new feature needs to find its layer, respect the boundaries, and communicate through the right channels. But the payoff is real:

- I can work on weather without touching NPCs
- New content types do not require changes across the codebase
- Sandboxes let me prototype fast without risk
- The CI lint catches violations before they reach main
- After months of development, the codebase still feels manageable

If you are building a larger game with Bevy, I would recommend thinking about your layer boundaries early. It does not have to be seven layers — the exact number depends on your project. What matters is having clear rules about what depends on what, and enforcing them.
