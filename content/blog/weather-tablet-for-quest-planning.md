+++
title = "A weather tablet for quest planning"
date = 2026-03-21
description = "Wild Spikes now has a weather tablet that shows a forecast, helping the player plan quests around dynamic weather conditions. Built as a standalone app and integrated through a separate crate."
[taxonomies]
tags = ["rust", "bevy", "wild-spikes", "weather", "ui"]
+++

Wild Spikes has dynamic weather. Rain, sun, clouds - the conditions change over time and affect the world. Some quests are easier to complete under certain weather conditions. Until now, the player had no way to know what was coming. That changes with the weather tablet.

![The weather tablet in Wild Spikes](/img/weather_tablet.png)

## What it does

The weather tablet shows a forecast of upcoming weather conditions. The player can pick it up, check what the next hours will look like, and plan accordingly. If a quest requires sneaking past NPCs and rain is coming - which reduces visibility - it might be worth waiting. If the sun is about to come out, it might be a good time to gather resources that only appear in clear weather.

It turns weather from a background system into something the player actively thinks about.

## How it is built

The tablet is split into two crates:

**weather_app** is a pure UI application that renders the forecast. It takes weather data as input and displays it. It has no knowledge of the game world, no physics, no 3D. It can run as a standalone Bevy app, which makes it easy to develop and test in isolation.

**weather_tablet** handles the integration into the game. It renders the app onto a display in the 3D world, manages colliders for interaction, and bridges the weather simulation data into the app. Everything that ties the tablet to the game lives here.

This separation follows the same layered architecture I described in my [previous post](@/blog/introduce-layers-into-bevy.md). The app sits in the engine layer - it is a generic, reusable UI component. The tablet crate sits in the game features layer, where it connects the app to the actual game systems.
