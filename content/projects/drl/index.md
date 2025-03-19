+++
title = "DRL (mod)"
updated = 2017-10-19
weight = 70
extra.role = "Developed mods in personal fork."
+++

{{ resize_image(
  path="/projects/drl/drl-screenshot.png",
  alt="DRL screenshot",
  width=640,
  height=480,
  op="fit"
) }}

DRL is a fast-paced run-and-gun roguelike by Kornel Kisielewicz about a soldier sent to stop an outbreak of hellish demons on a remote moon base.
The player wades into the chaos as one of three combat roles, gaining traits while increasing levels to unlock new perks and abilities.
An arsenal of brutal weapons and firearms can be tweaked with custom mod packs, or even transformed into new arnaments entirely.
The grunts and roars of demons that lurk in the moon base can be heard by direction against the backdrop of the game's heavy metal soundtrack.
All of this is topped off with a diverse system of medals, badges and multiple game modes.

DRL was featured on the fourth episode of the [Roguelike Radio](http://www.roguelikeradio.com/) podcast.

<!-- more -->

## My Personal Fork

I have a personal fork of the source code of this game that adds some quality-of-life features, interface improvements and bug fixes.
Some of the highlights include:

- A rework of the nerfed assault rifle assembly that increases damage while reducing ammo consumption and reload time.
- A wielded firearm can be reloaded from ammo directly on the ground at the player's feet, skipping the inventory.
- Drastic speed ups for mouse cursor movement in the graphical version of the game, and waiting in place in the ASCII version.
- Experimental fake terminal support with [libtcod](http://roguebasin.com/index.php/Doryen_library), with improved support for Esc, Ctrl, Alt and function keys.

[My personal fork of DRL on GitHub](https://github.com/tung/doomrl), and the [commit log](https://github.com/tung/doomrl/commits/patches) of my work with full details.

## More Information

- [DRL website](https://drl.chaosforge.org/)
- [DRL on RogueBasin](http://roguebasin.com/index.php/DoomRL)
