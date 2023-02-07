+++
date = 2023-02-07
title = "Making a CHIP-8 Interpreter: chip8run"
taxonomies.tags = ["c", "chip-8", "emulator", "programming", "webassembly"]
+++

I wanted to try my hand at making interactive graphical applications with C, WebAssemnbly and the [Sokol](https://github.com/floooh/sokol) header-only libraries when I stumbled across the [CHIP-8](https://en.wikipedia.org/wiki/CHIP-8).
Creating a CHIP-8 interpreter ("emulator") was as good an excuse as any to scratch my itch, and a little while later I ended up with [**chip8run**](https://tung.github.io/chip8run/), which you can try online right now.

<!-- more -->

## What is CHIP-8?

I suppose I'm getting ahead of myself.
CHIP-8 is a description of bytecode format for the creation of video games for home computers made in the 1970s and 1980s.
The idea is that you could write CHIP-8 bytecode as raw bytes to instruct a processor to do things with a 64-by-32 pixel monochrome display, while reading input from a 16-key hexadecimal keypad.
This was all that was needed to create simple video games that could run on CHIP-8 interpreters that were made for home computers such as the COSMAC VIP and DREAM 6800.

Nowadays, CHIP-8 still has games and demos written for it.
Here's some good links:

- [John Earnest's CHIP-8 Archive](https://johnearnest.github.io/chip8Archive/) - Lots of CHIP-8 games and demos.
- [Octo](http://johnearnest.github.io/Octo/) - An online CHIP-8 IDE.
- [OctoJam](http://octojam.com) - A CHIP-8 themed game jam.
- [Awesome CHIP-8](https://chip-8.github.io/links/) - Curated list of CHIP-8 resources.

## chip8run

Try chip8run online: <https://tung.github.io/chip8run/>

chip8run was created as an excuse to try using Sokol to create an interactive WebAssembly program.
I spent some time beforehand recreating part of the LearnOpenGL tutorials by [following along in geertarien's footsteps](https://www.geertarien.com/learnopengl-examples-html5/).
With some experience in hand, it didn't take long before chip8run emerged.

Sokol ended up being quite nice to work with overall.
The combination of pipelines, bindings and Sokol's shader cross-compiler made it easy to work with graphics hardware the way I wanted to.
The documentation in the header files and example code made it easy to add useful features like off-thread file loading and drag-and-drop support (try dragging a `.ch8` file into the browser version).

## Tips for Making a CHIP-8 Interpreter

Having done this, I have a few bits of advice for people looking to write their own CHIP-8 interpreter.

Check out the Awesome CHIP-8 link above and use [Timendus's CHIP-8 Test Suite](https://github.com/Timendus/chip8-test-suite) to slowly work through individual opcodes.
The readme of that test suite says how all the screens should work when everything is working properly.

When drawing sprites, modern CHIP-8 games expect both _x_ and _y_ wrapping; that is, sprites that intersect the right and bottom of the screen should have the rest of the sprite appear on the other side of the screen.
This technically deviates from the behavior of the CHIP-8 interpreter on the COSMAC VIP, but modern CHIP-8 games are often very buggy unless you do this.
If you're feeling ambitious, you can make this a quirk that can be toggled, otherwise opt for wrapping.

Some games have pretty intense flicker; chip8run deals with this by fading bright pixels out over time instead of immediately rendering them dark.
If you're making your own CHIP-8 interpreter, you may want to do the same.

Making a CHIP-8 interpreter is a pretty good way to try out a game library or framework.
John Earnest's CHIP-8 Archive linked above is a good place to find games and demos to try out once you want to move beyond testing.
