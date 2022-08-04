+++
title = "NetHack"
updated = 2016-04-26
weight = 30
extra.role = "Upstream bug fixes; developed mods in various personal forks."
+++

NetHack is a classic fantasy roguelike game; one of the most famous and venerable of its kind, with scores of contributors over a development history stretching back to its first release in 1987.
The player takes on the role of an adventurer sent on a divine quest to retrieve the fabled Amulet of Yendor, hidden deep in the firey pits of Gehennom below the perilous and ever-shifting Dungeons of Doom.

<!-- more -->

As a roguelike, NetHack continues in the biggest traditions of its genre.
*Permanent death* prevents players from saving and reloading their way out of trouble, forcing them to think on their feet.
Meanwhile, *procedural content generation* all but guarantees that no two journeys into the Dungeons of Doom are met with quite the same levels, monsters, items and situations.

NetHack is notable amongst roguelikes for a number of reasons:

- Players can choose amongst thirteen different roles, five races and three alignments.
- Hundreds of different items, many with unique magical effects.
- Hundreds of monsters, ranging from the lowly newt all the way up to angels and demon lords.
- Dozens of unique locations spanning castles, towns, swamps, seas, fortresses, temples and more.
- A rich and diverse ruleset governing how everything in the game interacts with everything else;
  for example, a player can blind a pursuing foe with a flash of light, and strike the drawbridge that they stumble onto with force bolt to plunge them into the waters below.
  A game of NetHack is as much an adventure with a story as it is hack-and-slash experience.

## Contributions

The current version of NetHack includes a couple of bug fixes that can be attributed to me:

- An important improvement to pathfinding for fleeing monsters in narrow corridors.
- A fix for confusion over the ownership of items in containers dropped by the player in a shop.

My NetHack variant [DynaHack](@/projects/dynahack.md) is credited in the official [NetHack Guidebook](https://nethack.org/v366/Guidebook.html) as one of the variants that helped maintain interest in NetHack during the long release cycle spanning NetHack 3.4.3 in 2003 and NetHack 3.6.0 in 2015.

## Personal Forks

I have a few forks of the NetHack source code where I've made modifications to the game.

### NAOHack Fork

Before the release of NetHack 3.6.0, the most-played version of NetHack was the one hosted on the alt.org public server.
Dubbed "NAOHack" by many players, this version was lightly patched relative to NetHack 3.4.3 to support server play, and could be considered a "polished" version of it.

I made a fork of NAOHack to add some quality-of-life improvements and fixes while still keeping the original NetHack 3.4.3 feeling:

- Dungeon map overview (press `Ctrl-O` to bring it up).
- Dark room patch.
- Colored walls and floor (enhanced).
- Autopickup ignores manually-dropped items.
- Autopickup exceptions can filter shop items.
- Auto-open doors.
- Auto-unlock for doors and containers.
- Auto-loot after auto-unlocking containers.
- `safe_peaceful` option to `#chat` to peaceful monsters by default instead of trying to attack them.
- `pilesize` option to reduce/suppress "Things that are here" window when walking on item piles.
- Fix travel command in Sokoban branch.
- Allow 'W'ear and 'T'ake off on rings, amulets and blindfolds (`wear_unified` option).
- Allow TTY text windows to be scrolled backwards (`<`, `>`, `^` and `|` navigation keys).
- Interrupt multi-turn actions when HP/Pw is restored, enhanced so running/travel ignore it.
- Stop running/travel when becoming hungry or weak from hunger.
- Fix for the infinite travel bug.

All of the above can be found on the [`patches` branch](https://github.com/tung/NAOHack/tree/patches) of [my NAOHack fork on GitHub](https://github.com/tung/NAOHack).

### NetHack 3.6.0 `statuscolors` Fork

NAOHack included a community patch called `statuscolors` that allowed players to add configurable colors the otherwise monochrome numbers and labels in the game's status area; this was often used to draw attention to it when the player was in danger.

When NetHack 3.6.0 was released, it shipped a similar feature that it called "Status Hilites".
Status hilites was unfortunately inferior to the community `statuscolors` patch: not only was its configuration syntax incompatible, it was unable to express highlighting criteria that `statuscolors` supported out-of-the-box.

This spurred me to create `statuscolors2`: a version of the `statuscolors` patch that not only worked with the newly-released NetHack 3.6.0, but also enhanced it in a number of ways:

- Full rule syntax compatibility with the previous `statuscolors` patch; useful for configuration files with many existing rules.
- Support all of the numeric status fields, not just "HP" and "Pw".
- Allow one-turn highlighting of numbers that change between turns.
- Brand new in-game customization menu, further enhanced with an export function to allow copying and pasting rules from the game to a configuration file.

I bundled this together with hit point bar coloring, culminating in the creation of the [Statuscolors and hitpointbar 2 patch](https://bilious.alt.org/?477) that I submitted to the NetHack Patch Database.
The NetHack Dev Team eventually improved their in-house status hilites system, and everybody updated their configuration files to match the new syntax, rendering this patch mostly redundant.
The development of this patch is preserved in this fork for historical reasons.

[My NetHack 3.6.0 fork with `statuscolors2` patch development on GitHub.](https://github.com/tung/nethack360-statuscolors/tree/statuscolors2)

### NetHack 3.6.0 Post-Release Patches Fork

I created a fork of NetHack 3.6.0 not long after its release to house enhancements and fixes that I felt were suitable for upstream inclusion.
Each change is contained in its own branch:

**`win-neutral-statuscolors-hitpointbar`**: The aforementioned Statuscolors and hitpointbar 2 patch in branch form.

**`tty-menu-enhancements`**: Prevent the `>` key from closing menus in the TTY windowport, so the `<` and `>` keys can be safely used to change menu pages.

**`tty-full-text-scroll`**: Allow full text windows, such as the message history and help windows, to be navigated backwards and forwards by page, instead of just forwards.
The `<`, `>`, `^` and `|` keys work like `PgUp`, `PgDn`, `Home` and `End` respectively in such windows.

**`attack-mode-option`**: Adds an `attack_mode` option with the following values and behaviors:

- `p - pacifist`: don't fight anything
- `c - chat`: chat with peaceful monsters, fight hostiles (default)
- `a - ask`: ask to fight peaceful monsters, fight hostiles
- `f - fightall`: fight peaceful and hostile monsters without asking

**`quest-artifact-stealing-fix`**: Stop the Wizard of Yendor from stealing the quest artifact of the player's current role.

**`map-coloring`**: Add thematic coloring for branches, special levels, rooms and dungeon features in the TTY windowport.
Adapted from [L's Coloured Walls and Floor v1 patch](http://bilious.alt.org/?296), but for NetHack 3.6.0.

**`general-main-daemon-fix`**: Fix mail daemons being randomly generated on special levels that place random demons;
the fix is generalized for any requested class of monsters that are all flagged to not be randomly spawned.

**`infinite-travel-fix`**: Fixes a bug where the automatic travel system will waste hundreds of turn oscillating while trying and failing to reach its destination.

[My NetHack 3.6.0 post-release patches fork on GitHub](https://github.com/tung/NetHack) and the [list of branches](https://github.com/tung/NetHack/branches/all) mentioned above.

## More Information

- [Official NetHack website](https://nethack.org/)
- [NetHack Wiki](https://nethackwiki.com/wiki/Main_Page)
- [Public NetHack server at alt.org](https://alt.org/nethack/)
- [Public NetHack server at Hardfought](https://www.hardfought.org/)
