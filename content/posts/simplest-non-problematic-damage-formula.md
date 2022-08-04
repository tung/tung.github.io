+++
date = 2022-08-02
title = "The Simplest Non-Problematic Damage Formula"
taxonomies.tags = ["game-design", "gamedev"]
+++

If you're designing a game with numeric hit points, attack and defense values, you'll eventually have to decide on a *damage formula* that determines the relationship between them.
The simplest approach possible is to calculate damage as just the difference between attack and defense; that is:

```
damage = attack - defense;
```

It's as simple as can be, but it has problems.
The above formula only produces satisfying damage values if attack is higher than defense.
Worse, damage will be zero if attack and defense are equal!
How can we fix this?

<!-- more -->

We can deal with the issue of attack needing to be much higher than defense by just doubling the attack value, like so:

```
damage = attack * 2 - defense;
```

The nice thing about this formula is that if the attack and defense values are equal, the damage value will be equal as well.

The issue with zero damage still exists, though now it happens when the attack value is half the defense value; as is, we're just kicking the can down the road.
As it turns out, there's a completely different damage formula that produces non-zero damage values for all non-zero attack values, while still reducing damage for increasing defense values.
Here it is:

```
damage = attack * attack / defense;
```

In the above formula, when attack and defense are equal, the damage value will be equal to the attack value.
The issue with this formula is that if the attack value becomes much higher than the defense value, the damage value will skyrocket!

The solution to this whole mess is simple: take the difference formula and the squaring formula, and combine them.
Behold, *the simplest non-problematic damage formula!*

```
if (attack >= defense) {
    damage = attack * 2 - defense;
} else {
    damage = attack * attack / defense;
}
```

We use the difference formula when attack is greater than or equal to defense since it doesn't blow up like the squaring formula.
When attack is low relative to defense, we switch to the squaring formula to all but guarantee that we get non-zero damage while attack itself is non-zero.{% note(num=true) %}If you're using integer division, attack values below the square root of defense can still round down to zero.{% end %}

You might argue that stitching two formulas together like this doesn't really count as *a* formula, and be tempted to search for or invent a single formula that solves everything.
Unfortunately, you won't find one, because *every formula breaks at its extremes*.
Even if you find or invent a single formula that seems good at first, it will often impose hard ranges of "good" values for attack and defense to avoid blowing up the damage value.
Special-casing damage formulas is the norm in all but the simplest damage systems in game design, so don't feel bad for reaching for branching logic.

Finally, feel free to garnish the damage value here.
For example, you can scale it up for skill levels, critical hits and weaknesses, or scale it down for resistances and situational defenses.
