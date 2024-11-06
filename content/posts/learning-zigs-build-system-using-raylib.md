+++
date = 2024-11-06
title = "Learning Zig's Build System using raylib"
taxonomies.tags = ["gamedev", "programming", "zig"]
+++

I picked up some Zig recently and figured I'd take the chance to learn its build system using raylib.
raylib already has pretty good community-maintained idiomatic Zig bindings, but we want to learn, so we're going to do it the "hard" way instead.

raylib has a `build.zig`, which means that it can be used directly with Zig's package manager.
Its C headers can also be translated into a Zig module that can be `@import`ed, thanks to Zig's built-in C translator.

Note: We will *not* be using `@cImport`, since it seems like it'll be removed in future versions of Zig.

<!-- more -->

## Warning about Versions

Zig, its build system and package manager, and Raylib's Zig support are all currently under active development.
Details are subject to change.

**Zig version**: This needs *Zig 0.14*, which is still in development and thus hasn't been released yet; Zig 0.13 will not work.

**raylib**: Commit `ad79d4a88422256d348ecfd06c53f8bc44b7777f` (2024-10-30); raylib v5.0 is too old for this.

NOTE: This version of Zig spits out a bunch of error messages when linking the final executable.
The build should still succeed regardless; check the output directory.

## Package Structure

The layout of files when we're done is going to look like this:

```
.
├── build.zig
├── build.zig.zon
└── src
    ├── c.h
    └── main.zig
```

`build.zig`: Package-specific build system logic; most of our focus and effort will go here.

`build.zig.zon`: Package manifest, including metadata and dependencies for Zig's package manager.

`src/c.h`: C header that will be translated into a Zig module that `main.zig` can `@import`.

`src/main.zig`: Zig code that uses raylib via the translated C header.

## The C Header

Put the following line of code into `src/c.h`:

```c
#include <raylib.h>
```

We'll arrange for this to be translated into Zig code that will let us use raylib without having to write any FFI bindings by hand.

## Zig Code that uses raylib

Put this into `src/main.zig`:

```zig
const c = @import("c");

pub fn main() void {
    c.InitWindow(800, 450, "raylib example");
    defer c.CloseWindow();

    c.SetTargetFPS(60);

    while (!c.WindowShouldClose()) {
        c.BeginDrawing();
        defer c.EndDrawing();

        c.ClearBackground(c.RAYWHITE);
        c.DrawText("This is a basic window.", 100, 200, 20, c.LIGHTGRAY);
    }
}
```

The C code in `src/c.h` will be turned into a module named "c" that we `@import` in this file.
C functions and definitions from raylib can be used directly from Zig like this.

## Filling in `build.zig.zon` for Zig's Package Manager

Put this into `build.zig.zon`:

```zig
.{
    .name = "example",
    .version = "0.1.0",
    .minimum_zig_version = "0.14.0",
    .dependencies = .{},
    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
    },
}
```

This file is in *Zig Object Notation*, which is just a cut-down version of Zig's struct syntax.
Here's the field-by-field break down:

`.name`: Package name that's used if this is a library; our's isn't, so we just put something sensible here.

`.version`: Package version; only really matters for library packages, which our's isn't.

`.minimum_zig_version`: Optional advisory field; we fill it with "0.14.0" here since our `build.zig` won't work with Zig 0.13.

`.dependencies`: List of Zig packages that Zig's package manager will get before building our package.

`.paths`: Files and directories of this package that Zig's package manager will keep after getting the source.

Our dependency list is empty right now.
Run the following command to fetch the raylib Zig package:

```
zig fetch --save=raylib https://github.com/raysan5/raylib/archive/ad79d4a88422256d348ecfd06c53f8bc44b7777f.zip
```

This outputs the following hash:

```
1220b990116595bb998d0ccb401b6497363321ec55e366402695cac531b6d93a1e34
```

This hash represents where the raylib dependency lives in the global Zig cache directory.
It is *not* a git hash.

If you look at `build.zig.zon` again, it should now look like this:

```zig
.{
    .name = "example",
    .version = "0.1.0",
    .minimum_zig_version = "0.14.0",
    .dependencies = .{
        .raylib = .{
            .url = "https://github.com/raysan5/raylib/archive/ad79d4a88422256d348ecfd06c53f8bc44b7777f.zip",
            .hash = "1220b990116595bb998d0ccb401b6497363321ec55e366402695cac531b6d93a1e34",
        },
    },
    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
    },
}
```

`raylib` is the dependency name that will be used later on in our upcoming `build.zig`.
A dependency can be a `url` and `hash` pair like this, or it can be a `path` instead.
If we wanted to use raylib for another package, we could copy-paste these dependency lines into some other `build.zig.zon`.
However, this will do for now.

With `build.zig.zon` filled in like this, raylib will be available as a dependency when building this package.
If this package is being built on another computer, raylib will be automatically fetched first via the URL.

## Zig Build System Concepts

Before moving onto filling in `build.zig`, we'll want to know a few foundational concepts.
Zig's build system is still in flux, so there isn't a lot of good high-level documentation.

The purpose of `build.zig` is to create a **build graph** that describes the actions and order of build-related tasks, such as compiling and linking our program.

The build graph is made up of **steps** that represent build actions.
Steps depend on other steps, forming a directed acyclic graph.

Steps often produce output files; Zig calls these **artifacts**.
Examples of artifacts are compiled and linked programs, libraries, and generated source and data files.

**Top-level steps** are special steps that serve as entry points into a build graph.
Top-level steps perform no actions; they only have other steps as depedencies.

The **`install` top-level step** exists in the build graph by default.
As the default top-level step, running `zig build` is equivalent to running `zig build install`.
You can add custom top-level steps to do whatever you want, such as `run` or `test`.

Unlike other tools like `make`, the inter-step dependencies of the build graph usually aren't listed directly.
Instead, you'll typically use helper functions that wire up most of the steps of the build graph internally.
Those helper functions often return values that can be fed into other helper functions to associate steps with one another.

## Describing the Package Build Graph in `build.zig`

Our upcoming `build.zig` has to create a build graph that does the following:

1. Read in and parse build-time options that customize the build.
2. Pull in the raylib dependency, made available thanks to our `build.zig.zon`.
3. Translate the C code in `src/c.h` into Zig bindings.
4. Create a Zig module from the translated `src/c.h` that links to raylib when imported.
5. Build `src/main.zig` that imports the C-to-Zig module into an executable.
6. Install the executable as part of the default `install` top-level step.
7. Create a step to run the executable, and a custom `run` top-level step to trigger it.

Create a `build.zig` with the following contents:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
}
```

Add all of the upcoming code samples inside the curly braces.

### Parse Build-Time Options

```zig
    const target = b.standardTargetOptions(.{});
```

This adds Zig's standard target options: `-Dtarget=`, `-Dcpu=` and `-Ddynamic-linker=`.
Zig is a batteries-included cross compiler and linker, with no special setup needed.

```zig
    const optimize = b.standardOptimizeOption(.{
        .preferred_optimize_mode = .ReleaseSmall,
    });
```

This adds Zig's standard optimize option, `-Doptimize=` that can currently be set to `Debug`, `ReleaseSafe`, `ReleaseFast` or `ReleaseSmall`.

Zig's build system also supports a special `--release=` flag that a user can employ to override any package-specific optimization choices.
It accepts `safe`, `fast` or `small` as values.
If no value is given, it'll use the `preferred_optimize_mode` value here by default.

If no optimization option is given at all to `zig build`, then `Debug` is the default optimization.

```zig
    const raylib_optimize = b.option(
        std.builtin.OptimizeMode,
        "raylib-optimize",
        "Override optimization of raylib",
    ) orelse optimize;
```

This parses a custom `--Doptimize-raylib=` option to let the user override the optimization of raylib.
It accepts the same `Debug`, `ReleaseSafe`, `ReleaseFast` and `ReleaseSmall` choices as the standard optimize option.

This command line flag is optional, and falls back on the package's optimization due to the `orelse` clause at the end.

### The raylib Depdency

```zig
    const raylib_dep = b.dependency("raylib", .{
        .target = target,
        .optimize = raylib_optimize,
    });
```

This pulls in the `raylib` dependency.
The `"raylib"` value here is the name as it appears in the `dependencies` list in `build.zig.zon`.

```zig
    const raylib_art = raylib_dep.artifact("raylib");
```

The Zig build system will build raylib from source, creating an artifact that we grab here.
We'll use it later to tell other steps where the raylib library and header files are located.
Those later uses will include the raylib dependency into the build graph, causing it to be built from source when needed.

### Translating `src/c.h` from C to Zig

```zig
    const translate_c = b.addTranslateC(.{
        .root_source_file = b.path("src/c.h"),
        .target = target,
        .optimize = optimize,
        .link_libc = true,
    });
    translate_c.addIncludePath(raylib_art.getEmittedIncludeTree());
```

This is our first real step.
`b.addTranslateC` creates a `std.Build.Step.TranslateC` which I'll call "a `TranslateC` step" for short.

The `root_source_file` points to the C source code to translate to Zig.
Using `b.path` specifies a path that is always relative to the package root directory.

The `target` and `optimize` fields are set according to the build-time options parsed earlier.

raylib uses the C standard library, so we set `link_libc` here to ensure that it's linked into the final executable.

Recall that `src/c.h` contains `#include <raylib.h>`.
For the `TranslateC` step to find `raylib.h`, the path where this dependency header file was installed must be added to the include path.
`raylib_art.getEmittedIncludeTree()` retrieves exactly that path.

### Creating a Zig Module from the Translated C Code

```zig
    const c_module = translate_c.createModule();
    c_module.linkLibrary(raylib_art);
```

This arranges for a module to be created from the translated C file.
Doing this allows it to be imported by Zig code.

We use `linkLibrary` to tell the build system to link to the raylib library when linking the final executable.

### Building the Executable

```zig
    const exe = b.addExecutable(.{
        .name = "example",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    exe.root_module.addImport("c", c_module);
```

`b.addExecutable` creates a `Compile` step that builds our executable.
`root_source_file` points at our package-relative `src/main.zig` file path.
`target` and `optimize` are specified as before.

`exe.root_module.addImport("c", c_module)` allows `@import("c")` to work inside `src/main.zig`.

### Installing the Executable

```zig
    b.installArtifact(exe);
```

This creates an `InstallArtifact` step internally that copies the final executable to the `zig-out/bin` directory.
This internal `InstallArtifact` step is added as a depedency of the default `install` top-level step, so running `zig build` will perform this action automatically.

Without this step, nothing will be built when `zig build` is run!

### Adding a Step to Run the Executable

```zig
    const run_exe = b.addRunArtifact(exe);
    run_exe.step.dependOn(b.getInstallStep());
    if (b.args) |args| run_exe.addArgs(args);
```

`b.addRunArtifact` creates a `RunArtifact` step that runs the executable as its action.

`run_exe.step.dependOn(b.getInstallStep())` is an example of making a step depend on another manually.
Usually this is done behind the scenes by helper functions, but this one is explicit.
This makes the `RunArtifact` step depend on the default `install` top-level step, so everything is installed before anything runs.
Technically this could be left out here, but this is handy if the executable expects other files to be installed before running.

The last line here allows forwarding command line arguments to the executable, e.g. `zig build run -- foo bar baz` will send `foo`, `bar` and `baz` to the executable as arguments.
This is also optional and could be left out here.

### A Custom `run` Top-Level Step

```zig
    const run_tls = b.step("run", "Run the app");
    run_tls.dependOn(&run_exe.step);
```

A `RunArtifact` step can't be invoked directly, since it could just be used internally, e.g. to generate files for the build.

`b.step` creates a top-level step that the user can pass to `zig build`.
`b.step("run", "...")` here enables `zig build run` to be recognized by the build system.
Despite the name, `b.step` specifically creates a *top-level step*, and not just any old step; the function name here is a bit confusing.

`run_tls.dependOn(&run_exe.step)` links this top-level step to the `RunArtifact` step that runs the executable from earlier.

That's it, we're done!

### The Complete `build.zig`

Here's the whole `build.zig` file in its entirety:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{
        .preferred_optimize_mode = .ReleaseSmall,
    });
    const raylib_optimize = b.option(
        std.builtin.OptimizeMode,
        "raylib-optimize",
        "Override optimization of raylib",
    ) orelse optimize;

    const raylib_dep = b.dependency("raylib", .{
        .target = target,
        .optimize = raylib_optimize,
    });

    const raylib_art = raylib_dep.artifact("raylib");

    const translate_c = b.addTranslateC(.{
        .root_source_file = b.path("src/c.h"),
        .target = target,
        .optimize = optimize,
        .link_libc = true,
    });
    translate_c.addIncludePath(raylib_art.getEmittedIncludeTree());

    const c_module = translate_c.createModule();
    c_module.linkLibrary(raylib_art);

    const exe = b.addExecutable(.{
        .name = "example",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    exe.root_module.addImport("c", c_module);

    b.installArtifact(exe);

    const run_exe = b.addRunArtifact(exe);
    run_exe.step.dependOn(b.getInstallStep());
    if (b.args) |args| run_exe.addArgs(args);

    const run_tls = b.step("run", "Run the app");
    run_tls.dependOn(&run_exe.step);
}
```

The Zig build system can be invoked in the following ways:

- `zig build` for a Debug build; this ends up in the `zig-out/bin` directory.
- `zig build --release` for the default ReleaseSmall build.
- `zig build run` to build and run the Debug build.
- `zig build run --release` to build and run the ReleaseSmall build.

## Wrapping Up

Summing everything up, we:

1. Created a module from C code in order to use raylib in a Zig program.
2. Made a `build.zig.zon` manifest to fetch raylib as a dependency.
3. Wrote a `build.zig` to build raylib, translate the C code into a module, build our Zig program and link them all together.

The Zig build system and pacakage manager duo can be somewhat intimidating at first; the lack of documentation so far definitely doesn't help.

The `build.zig` file for this package is larger than usual due to having to translate C into a Zig module.
Pure Zig projects typically have shorter build specifications.

If you want to do things with the Zig build system, try to find examples by Andrew Kelley, the Zig creator.

If you want to go further than the examples, remember that the build helper functions return values with types you can feed into other helper functions.
Look at the `std.Build` section of the standard library docs.
Link up return values and parameter types: all the steps, artifacts, modules, dependencies and `LazyPath`s will often narrow down your options.
Those docs also link to implementation source code that you may want to consult.

Hopefully official documentation will make scavenging types and source code like this less necessary in the future.
