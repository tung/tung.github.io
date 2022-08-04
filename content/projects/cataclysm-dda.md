+++
title = "Cataclysm: Dark Days Ahead (fork)"
updated = 2020-09-07
weight = 60
extra.role = "Upstream bug fixes; developed mods in personal fork."
+++

Cataclysm: Dark Days Ahead is a turn-based survival game set in a post-apocalyptic world.
Players must explore and survive while defending against zombies and countless other monstrosities in a limitless, procedurally-generated world.
Items can be scavenged and crafted; structures built; farms cultivated; vehicles repaired and modified.
The player can install advanced bionic enhancements, and even mutate.

<!-- more -->

## Upstream Bug Fixes

Cataclysm: Dark Days ahead is a highly-active open source project with hundreds of contributors over the years.
My contributions to the upstream project are fairly minor:

- I traced the source of the mystery of fresh ingredients cooking into rotten food to an underflow error during calculations involving unsigned integers.
- I suggested the fix to reduce lightstrip duration from years (!!!) to days.

## My Personal Fork

I have a personal fork of Cataclysm: Dark Days Ahead, based on version 0.E stable, before the inclusion of nested containers and mandatory Z-levels.
I've made over a hundred individual changes to the game to tweak it to how I like it to be played; here are some of the highlights:

- Greatly improved support for bitmap fonts, particularly for anti-aliasing and blending.
  This allows players to bypass the game's in-built font rendering and replace it with their own choices, right down to the pixel.
- Allow household item density to be separately configured; reduce it for a more scavenging-focused game!
- Reduce massive abundance of ammo from turrets and freely-available fresh water from forests and swamps.
- Fix overly-tight limits on aiming speed based on firearm volume so that gun mods that improve aiming speed can actually do so.
- Add 'S' key to armor sorting screen to reposition armor by layer.
- Allow reviving corpses to be sorted at the top of the visible items sidebar list.
- Fix 'ghost' trails of dragged vehicles in map memory.
- Dozens of minor user interface and quality-of-life tweaks.

[My personal fork of Cataclysm: Dark Days Ahead on Github](https://github.com/tung/Cataclysm-DDA), and the [commit log](https://github.com/tung/Cataclysm-DDA/commits/custom) with all of my changes.

## More Information

- [Cataclysm: Dark Days Ahead website](https://cataclysmdda.org/)
- [Cataclysm: Dark Days Ahead source code on GitHub](https://github.com/CleverRaven/Cataclysm-DDA/)
