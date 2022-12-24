+++
date = 2022-12-24
title = "My Workflow for following Crafting Interpreters"
taxonomies.tags = ["c", "linux", "make", "programming", "testing"]
+++

I recently worked my way through Robert Nystrom's [_Crafting Interpreters_](https://craftinginterpreters.com), which guides you through the process of making an interpreter for a scripting language.
The book is made up of three parts: a short part describing Lox (the scripting language made for the book), a tree-walk interpreter written in Java and a bytecode virtual machine interpreter written in C.
This post is about how I approached the last part.

The book doesn't recommend any specific setup or workflow, so you're free to go with whatever you like.
The rest of this post describes what I went with; all of the code can be found here: <https://github.com/tung/clox>

## Goals

_Crafting Interpreters_ has every line of code needed to create _clox_, the C bytecode VM for the Lox language, and it'd be easy to just type it all in and nod along.
However, I wanted to go a little further, so I set a couple of extra goals for myself:

- Come up with a good testing workflow.
- Learn about and integrate tools to support code quality, e.g. <abbr title="Language Server Protocol">LSP</abbr> server, automatic formatting, code coverage.

## The Setup

I run Linux and prefer a command-line-centered approach for small-to-medium-sized projects like these, so my basic setup looked like this:

- C compiler - **GCC**.
- Build system - **GNU Make** with some **GNU Bash** to back it up.
- Code editor - [**Helix**](https://helix-editor.com), an up-and-coming editor with out-of-the-box LSP support.
- Version control - **Git**.

Along the way I pulled some extra tools into the fray:

- LSP server - [**clangd**](https://clangd.llvm.org/), which Helix picks up right away.
- `compile_commands.json` generator for the LSP server - [**Bear**](https://github.com/rizsotto/Bear).
- Automatic code formatter - [**clang-format**](https://clang.llvm.org/docs/ClangFormat.html).
- Code coverage report generator - [**gcovr**](https://gcovr.com).

## Editing Code with Helix

[**Helix**](https://helix-editor.com) is a modern modal text editor with out-of-the-box LSP support and [Tree-sitter](https://tree-sitter.github.io/tree-sitter/)-powered highlighting and navigation.
As a long-time Vim user, it took me a couple of weeks to get used to it, but it was refresing to use an editor without spending hours figuring out how to integrate external tools I wanted to use.
The LSP support highlights warnings and errors as you type so you can deal with them right away.

I never thought setting up fuzzy file opening was worth the hassle in Vim, but I found pressing `Space+f` to pick files and `Space+b` to switch between open buffers to be quite convenient in Helix.

LSP-powered code navigation was handy, particularly when tackling challenges where a certain amount of refactoring was called for.
Pressing `gd` (go to definition) takes you to the definition of a symbol; from there `Ctrl+o` (out) and `Ctrl+i` (in) act like back and forward in a web browser, but for code locations.
For this to work properly you need a `compile_commands.json` file; I use [**Bear**](https://github.com/rizsotto/Bear) for this whenever a source file is added or renamed:

```sh
$ make clean && bear make all
```

Helix doesn't have a visual block mode like Vim yet, but you can get a similar effect using its _multiple cursors_ feature.
Pressing `C` creates an additional cursor on the next line, so you can line up the cursor at the first line of a block, press `C` a bunch of times and type on every line simultaneously;
this is pretty handy for dealing with test data and quickly commenting and uncommenting code blocks.

## Compiling and Linking with GNU Make

The two biggest jobs of a Makefile for a C project are:

1. Compile each C source file (ending with `.c`) into a corresponding object file (ending with `.o`).
2. Link the object files into executable binary files.

There's a lot of information out there about how to write simple Makefiles that do these two things, but they have a lot of downsides:

- Object files are generated next to source files, which makes navigating the source code messy and complicates cleaning.
- Compiling source files with different flags requires any previous build to be cleaned out first, e.g. for debug versus release builds.
- The list of source files each object file depends on needs to be kept up-to-date when files change.
- The list of object files each binary depends on needs to be kept up-to-date when files change.

Tools like CMake and GNU Autotools can deal with or mitigate some of these issues, but they're also pretty complex.
The clox interpreter as described in _Crafting Interpreters_ consists of twenty source files with no external dependencies, so I decided to tackle all of this with just GNU Make instead.
With a bit of elbow grease, I ended up with a Makefile that I didn't have to update, even when new files were introduced or `#include`s were changed.

The final Makefile: <https://github.com/tung/clox/blob/master/Makefile>

### Out-of-Tree Builds

The first issue with a simple C Makefile is that object files end up in the same directory as the source code.
Right away, I decided on the directory layout I wanted files to end up in:

- `src/` for C source files _and_ headers
- `build/` for build artifacts like object files and binaries

I decided against having a separate `include/` directory.
Project-local headers are typically `#include`d with double quotes, which take file-relative paths and only fall back on searching include paths afterwards, so it's simpler to `#include` such files when they live in the same directory.
On top of this, a header named `foo.h` is most likely to change when its matching `foo.c` source file changes; keeping those files next to each other makes it easier to open one when changing the other.

The Makefile defines explicit variables for these directories:

```make
src_dir = src/
header_dir = $(src_dir)
build_dir = build/
```

Having explicit variables like this is useful later on, so there's no mucking about with `VPATH` or anything like that.

If all I wanted was to split out build artifacts from source files, I could define Make rule targets directly in terms of `build_dir`.
But it doesn't end here, because I wanted...

### Build Modes

I wanted to support compiling and linking clox in at least two different ways:

1. _Debug mode_ with debugging information and extra runtime checks.
2. _Release mode_ with all optimizations applied.

Therefore, each source file compiles into a least _two_ separate object files, which are linked into _two_ separate `clox` interpreter binaries.
In other words, I needed a `build/debug/` directory for debug build artifacts and a `build/release/` directory for release build artifacts.

The Makefile starts by taking a `MODE` variable{% note(num=true) %}[By convention](https://www.gnu.org/software/make/manual/make.html#toc-How-to-Use-Variables), upper case names are used for variables that can be overriden.{% end %} that can be set to `debug` or `release`, defaulting to the former, and defining a `mode_dir` variable:

```make
ifeq ($(MODE),)
  override MODE = debug
  $(info Using default MODE = $(MODE).)
endif

mode_dir = $(build_dir)$(MODE)/
```

The `mode_dir` is set to a subdirectory of `build_dir`, and it's where all of the files created by the build system will be placed.
Thus, all files created by the build are defined in terms of this `mode_dir`.

The `MODE` also determines the values of the `CFLAGS`, `LDFLAGS` and `LDLIBS` variables.{% note(num=true) %}These variables are used by [GNU Make's implicit rules](https://www.gnu.org/software/make/manual/make.html#Catalogue-of-Rules), but otherwise there's nothing special about them.{% end %}
First, for debug mode:

```make
ifeq ($(MODE),debug)
  CFLAGS = -Wall -Wextra -Og -g -fsanitize=address -fsanitize=undefined -fno-omit-frame-pointer
  LDFLAGS = -Og -g -fsanitize=address -fsanitize=undefined
  LDLIBS = -lm
```

The `LDLIBS = -lm` is needed for some C standard library math functions I eventually ended up needing, but the interesting part is the other two lines.
A break-down of the flags I chose for debug mode:

- `-Wall -Wextra` - Turns on lots of warnings to ward off problems before they become bugs.
  Some people add `-Werror` to this mix, but I chase down all warnings on sight, so I didn't need it.
- `-Og -g` - Include debugging information; `-g` adds the info, and `-Og` runs "optimization" passes that gather extra debugging info.
- `-fsanitize=address` - Turns on [**AddressSanitizer**](https://clang.llvm.org/docs/AddressSanitizer.html) to detect memory errors;
  this is _extremely_ helpful when working on the [garbage collection chapter](https://craftinginterpreters.com/garbage-collection.html).
  Any memory errors will produce stack traces where the memory error occurred, where it was freed and and where it was allocated.
  This flag automatically enables `-fsanitize=leak` for [**LeakSanitizer**](https://clang.llvm.org/docs/LeakSanitizer.html) on Linux, which catches unfreed memory on exit.
- `-fno-omit-frame-pointer` - Keeps frame pointers for better stack traces.

AddressSanitizer works very nicely when running the garbage collector in stress mode (i.e. collecting before every single memory allocation);
doing so will catch and precisely identify memory allocations you have in the stack but forgot to keep rooted.

Here's release mode:

```make
else ifeq ($(MODE),release)
  CFLAGS = -Wall -Wextra -O3 -flto -march=native
  LDFLAGS = -O3 -flto -march=native
  LDLIBS = -lm
```

Release flags break-down:

- `-O3` - Optimize for performance.
- `-flto` - Link-time optimization.
  The way it works is by first compiling each source file into an compiler-internal representation.
  When everything is linked, optimizations are applied as if everything had been part of the same source file.
- `-march=native` - Allows the use of features of the CPU currently being used to build the project.
  It reduces portability, but boosts performance a bit.

### Automatic Dependencies for Compiling C Files

GCC's C preprocessor can output the list of source and header files that an object file depends on.
For example, I can get the dependencies of `scanner.o` by processing `scanner.c` like so:

```sh
$ cc -E -MM -MT build/debug/objs/scanner.o scanner.c
build/debug/objs/scanner.o: src/scanner.c src/scanner.h src/common.h
```

`cc -E` runs the C preprocessor, `-MM` emits the Make-compatible rule with dependency info (no system headers) and `-MT foo` sets the rule target, i.e. the part before the `:` (colon) character in the output.

GNU Make can then include this Makefile fragment{% note(num=true) %}I call these "Makefile fragments"; I don't know if they have an official name.{% end %} and roll it into its internal dependency graph, so object files are only rebuilt when the _exact_ files they depend on are changed.

The cut-down setup put together looks like this:

```make
# From before.
src_dir = ...
build_dir = ...
mode_dir = ...

# e.g. build/debug/deps/
deps_dir = $(mode_dir)deps/

c_srcs = $(wildcard $(src_dir)*.c)
c_deps = $(c_srcs:$(src_dir)%.c=$(deps_dir)%.d)

# Generate *.d files from *.c files.
$(c_deps): $(deps_dir)%.d: $(src_dir)%.c
	$(CPP) $(CPPFLAGS) -MM -MT $(@:$(deps_dir)%.d=$(objs_dir)%.o) -MT $@ -MP -MF $@ $<

# Read in *.d files; generate them if they don't exist yet.
-include $(c_deps)

# e.g. build/debug/objs/
objs_dir = $(mode_dir)objs/

c_objs = $(c_srcs:$(src_dir)%.c=$(objs_dir)%.o)

$(c_objs): $(objs_dir)%.o: $(src_dir)%.c
>$(CC) $(CPPFLAGS) $(CFLAGS) -c -o $@ $<
```

There's some Makefile voodoo going on up there, which I'll try to explain.

`$(wildcard src/*.c)` is a bit like running `ls src/*.c`, so `c_srcs = $(wildcard $(src_dir)*.c)` sets the `c_srcs` variable to something like `src/foo.c src/bar.c src/baz.c`.

`$(c_srcs:$(src_dir)%.c=$(deps_dir)%.c)` is a [substitution reference](https://www.gnu.org/software/make/manual/make.html#Substitution-Refs) that expands to something like `$(c_srcs:src/%.c=build/debug/deps/%.d)`, which matches each file in `c_srcs` against the pattern `src/%.c` and replaces the non-`%` parts with the pattern `build/debug/deps/%.d`.
For example, going with `c_srcs` from before, this would come out to `build/debug/deps/foo.d build/debug/deps/bar.d build/debug/deps/baz.d`.

The `$(c_deps): $(deps_dir)%.d: $(src_dir)%.c` is the first line of a [static pattern rule](https://www.gnu.org/software/make/manual/make.html#Static-Pattern).
If it were typed manually, it'd expand to this:

```make
build/debug/deps/foo.d: src/foo.c
	...

build/debug/deps/bar.d: src/bar.c
	...

build/debug/deps/baz.d: src/baz.c
	...
```

The tab-indented lines are the _recipe_ of the rule.
The `$@` and `$<` are references to the [automatic variables](https://www.gnu.org/software/make/manual/make.html#Automatic-Variables) `@` and `<`;
`@` is a stand-in for the rule _target_ (the thing to the left of the `:` character), while `<` is the first prerequisite (the first thing to the right of the `:` character).
For example, we can expand the rule to make the target `build/debug/deps/foo.d` like so:

```make
build/debug/deps/foo.d: src/foo.c
	$(CPP) $(CPPFLAGS) -MM -MT build/debug/objs/foo.o -MT build/debug/deps/foo.d -MP -MF build/debug/deps/foo.d src/foo.c
```

`-MT` can be specified multiple times to put multiple things to the left of the `:` character when the C preprocessor works its magic;
the `foo.d` Makefile fragment is included so the dependency will rebuild itself along with the object file if any of the discovered dependencies change.
Meanwhile, `-MP` creates phony rules to deal with errors that Make complains about when files are deleted and `-MF foo.d` writes the Makefile fragment to the file named `foo.d`.

The `-include $(c_deps)` line is where all the magic happens.
The `include` keyword causes GNU Make to [include our Makefile fragments](https://www.gnu.org/software/make/manual/make.html#Include);
using `-include` (with the leading dash) instead ignores warnings about missing files.
If GNU Make realizes these files are missing, it'll finish reading the Makefile and run the rules needed to create them before restarting and running itself again.{% note(num=true) %}There's a [limit](https://www.gnu.org/software/make/manual/make.html#Remaking-Makefiles) to the number of times GNU Make will restart itself, so there's no need to worry about infinite loops.{% end %}

The rules to build the object files from the C source files are typical fare for Makefiles.
This takes advantage of the fact that a target can have multiple rules that can build up the dependency list, as long as only one of them has recipe.

### Automatic Dependencies for Linking C Programs

If I was only going to link all the object files into a single binary, a simple rule like this would be enough (where `^` is the dependency list for the rule):

```make
# From before.
c_objs = ...

clox: $(c_objs)
	$(CC) $(LDFLAGS) -o $@ $^ $(LDLIBS)
```

However, I knew I would have additional binaries for things like tests and benchmarks, and each of those would need their own lists of object files.
Managing object dependencies for linking test and benchmark binaries would be tedious and discourage me from adding new ones when I needed to, so I wondered if there was a way to work them out automatically.
As it turns out, there is, but it takes some creative thinking.

The goal here is to discover the object files (ending with `.o`) needed to link a binary, and do so automatically.
I needed to do what the C preprocessor was doing for object files reading source and header files, but do it for binaries with object files.

The key is in the Makefile fragments from before (ending with `.d`): if you squint, you can see that it describes the entire dependency graph!
Specifically, if a Makefile fragment mentions a `.o` file, it needs to be linked in, and if it mentions a `.h` file, it probably has a matching `.c` file and therefore a `.d` file that we should also read.
The approach looks roughly like this:

1. Initialize a `.d` list with the `.d` file for the `.c` source file containing the `main` function.
2. While the `.d` list has more entries:
   - If the current `.d` file doesn't exist, skip to the next in the `.d` list.
   - Read space-separated words from the current `.d` file.
   - If the word ends with `.o`, add it to the list of objects to link into the binary if it's not already there.
   - If the word ends with `.h`, convert it to into a `.d` file path and add it to the `.d` list if it's not already there.
3. Output a Makefile-compatible rule with the binary name as the target and the list of objects as dependencies.

Here's the Bash script I wrote that does this: <https://github.com/tung/clox/blob/master/linkrule.bash>

The rule to create automatic link dependencies for the main clox interpreter binary looks like this:

```make
# From before.
src_dir = ...
header_dir = ...
deps_dir = ...
mode_dir = ...
c_deps = ...

# Directory of this Makefile; not fully reliable, but good enough for our purposes.
makefile_prefix = $(dir $(firstword $(MAKEFILE_LIST)))

main_name = clox
main_src = $(src_dir)main.c
main_target = $(mode_dir)$(main_name)

# e.g. build/debug/deps/main.link.d
main_link_dep = $(main_src:$(src_dir)%.c=$(deps_dir)%.link.d)

# Generate *.link.d files from *.d files.
$(main_link_dep): | $(c_deps)
	$(makefile_prefix)linkrule.bash $@ $(main_target) $(@:.link.d=.d) $(header_dir)

# Read in e.g. build/debug/deps/main.link.d; generate it if it doesn't exist yet.
-include $(main_link_dep)

# Link the main target binary, e.g. build/debug/clox
$(main_target):
	$(CC) $(LDFLAGS) -o $@ $^ $(LDLIBS)
```

The `$(main_link_dep): | $(c_deps)` line is new;
the `|` (pipe) character denote [order-only prerequisites](https://www.gnu.org/software/make/manual/make.html#Prerequisite-Types) that are only checked for existence, not timestamp like normal prerequisites.
If the `.d` files don't exist yet, I want all of them to be created before reading anything.
After that, the `linkrule.bash` script contains more fine-grained dependency information for specific `.d` files.

The `$(@:.link.d=.d)` substitution reference matches the `.link.d` suffix and replaces it with just a `.d` suffix.

If `clox` was the only binary, this would be a huge waste of time.
Doing this for test binaries is a lot more useful:

```make
# From before.
src_dir = ...
header_dir = ...
deps_dir = ...
mode_dir = ...
c_deps = ...
makefile_prefix = ...

# e.g. src/foo_test.c src/bar_test.c
test_srcs = $(wildcard $(src_dir)*_test.c)

# e.g. build/debug/foo_test build/debug/bar_test
tests = $(test_srcs:$(src_dir)%.c=$(mode_dir)%)

# e.g. build/debug/deps/foo_test.link.d build/debug/deps/bar_test.link.d
test_link_deps = $(test_srcs:$(src_dir)%.c=$(deps_dir)%.link.d)

# Generate *.link.d files from *.d files for tests.
$(test_link_deps): %.link.d: | $(c_deps)
	$(makefile_prefix)linkrule.bash $@ $(*:$(deps_dir)%=$(mode_dir)%) $(@:.link.d=.d) $(header_dir)

# Link the test binaries, e.g. build/debug/foo_test and build/debug/bar_test.
$(tests): %:
	$(CC) $(LDFLAGS) -o $@ $^ $(LDLIBS)
```

The `*` (asterisk) automatic variable above refers to the part of the static pattern rule that the `%` matches, e.g. the following echoes `build/debug/deps/foo_test`:

```make
build/debug/deps/foo_test.link.d: %.link.d: | $(c_deps)
	echo $*
```

The `$(*:$(deps_dir)%=$(mode_dir)%)` therefore turns `build/debug/deps/foo_test.link.d` into something like `build/debug/foo_test`, which is the output file path I want for the test binary.

When this was all put together, it closed the Makefile maintenance loop: I never had to revisit the Makefile when adding or moving files around at all!
Having this setup made working with C feel a bit more like working with a modern language.

### Bonus: Including Git Version Information

I thought it'd be nice for the clox binary to print out its version on request, like this (where `dedbeef` would be the short Git revision hash):

```sh
$ ./build/debug/clox --version
clox debug-git-dedbeef
```

I decided to have the Makefile generate this information as a Makefile fragment and rebuild `src/main.c` to include it.

The idea is to have the Makefile run `git describe --always --dirty` and compare it to a Makefile fragment with the output of the same command from the previous run.
If the output and the fragment disagree, delete the fragment so it's considered out-of-date and recreated.
Object files created from `src/main.c` are set to depend on that fragment file, so they'll also be rebuilt.
Finally, the Makefile fragment is `-include`d and the `CFLAGS` is extended with that version string, but only for those object files.
Here it is, in stripped-down form:

```make
# From before.
MODE = ...
src_dir = ...
mode_dir = ...
objs_dir = ...
c_objs = ...
main_obj = ...

# e.g. build/debug/version.mk
version_mk = $(mode_dir)version.mk

# Produce a version string for the current git repo state, e.g. debug-git-dedbeef
version = $(MODE)$(shell V=$$(git -C $(src_dir) describe --always --dirty 2>/dev/null) && printf -- "-git-%s" "$${V}" || printf "")

# e.g. "version_str = debug-git-dedbeef"
version_fragment = version_str = $(version)

# Create the version Makefile fragment.
$(version_mk):
	printf "%s\n" $(version_fragment) > $(version_mk)

# If past and present version info differ, delete the fragment so it's recreated.
ifneq ($(file <$(version_mk)),$(version_fragment))
  $(shell $(RM) $(version_mk))
endif

# Include the fragment so we can read the version_str variable.
-include $(version_mk)

# Rebuild e.g. build/debug/main.o if build/debug/version.mk is recreated.
$(main_obj): $(version_mk)

# Add e.g. -DVERSION="debug-git-dedbeef" to CFLAGS, but only for build/debug/main.o.
$(main_obj): CFLAGS += -DVERSION="$(version_str)"

# Updated static pattern rule to compile *.o files from *.c files.
$(c_objs): $(objs_dir)%.o: $(src_dir)%.c
	$(CC) $(CPPFLAGS) $(CFLAGS) -c -o $@ $(filter-out $(version_mk),$<)
```

The line that runs `git describe` looks funny, but it ensures that the `version` variable is suffixed by either `-git-dedbeef` (where `dedbeef` is the revision hash) or an empty string.

The `file` function reads in any previous version fragment and compares its contents to that of the current run.
If they differ, the file is deleted;
this needs to be done before the `-include` line so that GNU Make realizes that the file needs to be recreated.

The rule to compile `.o` files from `.c` files is updated to ensure that the Makefile fragment isn't used in the compiler command line.

With all of this in place, `src/main.c` can print the version info from the `VERSION` definition given to it like so:

```c
#include <stdio.h>

#define STRINGIFY2(x) #x
#define STRINGIFY(x) STRINGIFY2(x)

static void printVersion(void) {
  puts("clox " STRINGIFY(VERSION));
}
```

This `printVersion` function can then be called from the `main` function when `--version` is passed as a command line flag.

## Automatic Code Formatting

I used [**clang-format**](https://clang.llvm.org/docs/ClangFormat.html) to format code;
even though I worked on this alone, I didn't want to waste time thinking about how to split long lines of code.

The first step to using it is to create a `.clang-format` file:

```sh
$ clang-format -style=llvm -dump-config > .clang-format
```

I tweaked this file a bit to set the maximum line length I wanted, and adjust how long lines should be broken across multiple lines.

From here, I created two task targets in my Makefile.
The first reformats source files in-place:

```make
# From before.
header_dir = ...
c_srcs = ...

# Third-party code that shouldn't be formatted.
no_format_srcs = ...

# Source files to format.
format_srcs = $(filter-out $(no_format_srcs),$(c_srcs) $(wildcard $(header_dir)*.h))

.PHONY: format
format:
	clang-format -i $(format_srcs)
```

I'd use this by editing code as usual until I was ready to commit and `git add` those changes.
I'd then run `make format` to reformat the code, review the changes using `git diff`, and tweak things as needed before running `git add` again and committing the formatted code.

The second task checks if the code formatting is okay and exits with an error if it isn't:

```make
# From before.
format_srcs = ...

.PHONY: checkformat
checkformat:
	clang-format --dry-run -Werror $(format_srcs)
```

I wanted to use this to stop unformatted code from being committed, so I wrote this `git-pre-commit.sh` script:

```sh
#!/bin/sh
set -eu

STASH_NAME="pre-commit-$(date -Iseconds)"
git stash push --keep-index --include-untracked "${STASH_NAME}"

make checkformat

if [ "$(git stash list | head -n 1)" = "*${STASH_NAME}" ]; then
  git stash pop
fi
```

The functional part of this script is bracketed with `git stash push --keep-index ...` and `git stash pop` to ensure that only the code to be commited is checked by `make checkformat`.

This script is installed as a pre-commit hook for git like so:

```sh
$ ln -s ../../git-pre-commit.sh .git/hooks/pre-commit
```

This setup checks that all the code is formatted before allowing it to be committed.

## Testing

As mentioned earlier, I really wanted to write tests for the code I was writing.
I looked at a few C testing frameworks before settling on [**utest.h**](https://www.duskborn.com/utest_h/), a single-header testing framework.

If I wanted unit tests for a file named `src/foo.c`, I'd create a matching `src/foo_test.c` file, `#include "utest.h"` at the top and write my test code.
The `_test.c` suffix would be picked up automatically by the Makefile to produce a test binary at e.g. `build/debug/foo_test` that could be launched to run its tests.

Aside from these unit tests, I also created `src/interpret_test.c`, an integration test that sent input strings through the whole clox scanning/parsing/compiling/running machinery and tested for outputs and errors.
This is pretty much the most important test file in the repository, since it tests for desired end-to-end behaviors that the unit tests individually do not.

All of the tests are run via this Makefile task:

```make
# From before.
tests = ...
makefile_prefix = ...

.PHONY: test
test: $(tests)
	$(makefile_prefix)runtests.bash $(tests:%=./%)
```

Running `make test` would run all the tests through my `runtests.bash` shell script, which takes the test binaries as arguments, runs them all and prints the names of any tests that fail.

The `runtests.bash` script: <https://github.com/tung/clox/blob/master/runtests.bash>

## Code Coverage

Having testing up and running is all fine and well, but how would I know what to test for?
With _code coverage_ instrumentation, I could run the tests in a special "coverage" mode, then process the coverage data into a report to check how much and which parts of the code were actually being tested.

I didn't really want to constantly produce coverage data, so I created a new `coverage` mode alongside the existing `debug` and `release` modes:

```make
# Updated with "coverage".
all_modes = debug release coverage

# ...
else ifeq ($(MODE),coverage)
  CFLAGS = -Wall -Wextra -Og -g -fsanitize=address -fsanitize=undefined -fno-omit-frame-pointer --coverage
  LDFLAGS = -Og -g -fsanitize=address -fsanitize=undefined --coverage
  LDLIBS = -lm
```

The new flag here is `--coverage`, which adds coverage instrumentation when added to both `CFLAGS` and `LDFLAGS`.
This produces `.gcno` files in the same directory as the `.o` files when `.c` files are compiled.
When the binaries linked with these `.o` run, coverage data accumulates in `.gcda` files in that same directory.

The coverage data in the `.gcda` files can be processed into a multi-file HTML report with the help of [**gcovr**](https://gcovr.com).
I worked it into the Makefile like this:

```make
# BEFORE setting the default mode to "debug".
ifeq ($(MAKECMDGOALS),gcovr)
  override MODE = coverage
  $(info MODE = coverage override for gcovr target.)
endif

# ...

# From before.
mode_dir = ...
objs_dir = ...

# e.g. build/coverage/gcovr/
report_dir = $(mode_dir)gcovr/

# e.g. build/coverage/gcovr/report.html
report_loc = $(report_dir)report.html

# Tests, benchmarks and third-party code to exclude from the coverage report.
report_exclude = ...

# Create the report directory.
$(report_dir): | $(mode_dir)
	mkdir $@

# Generate code coverage report from coverage data.
$(report_loc): $(objs_dir) $(wildcard $(objs_dir)*.gcda) | $(report_dir)
	gcovr $(report_exclude:%=-e %) --html-details $(report_loc) -r $(src_dir) $(objs_dir)

# Print a link to the report once it's generated.
.PHONY: report
report: $(report_loc)
	@printf "Coverage report at: file://%s\n" $(abspath $(report_loc))

# Delete report and reset accumulated .gcda coverage data.
.PHONY: cleancoverage
cleancoverage:
	$(RM) -r $(report_dir)
	$(RM) $(objs_dir)*.gcda

# Run tests with coverage, generate report and show its link.
.PHONY: gcovr
gcovr: cleancoverage
	$(MAKE) -f $(firstword $(MAKEFILE_LIST)) MODE=coverage test
	$(MAKE) -f $(firstword $(MAKEFILE_LIST)) MODE=coverage report
```

Running `make gcovr` with this in place resets everything, builds and runs all of the tests in coverage mode, then produces a multi-file HTML report using gcovr.
Code that is exercised is highlighted in green, while unexercised lines are highlighted red;
I used this as a guide for what to test for.

When running `gcovr`, using `-e foo.c` excludes `foo.c` from the report.
The `--html-details` option picks the multi-file HTML report format that I want.
The `-r` option affects the path that source files mentioned in the report are relative to;
I point it right at the source directory for covenience.
The final option is where gcovr should look for the `.gcno` and `.gcda` files to produce its report.

## Benchmarking

Benchmarks have a very similar setup to tests in the Makefile, except I write them with the help of [**ubench.h**](https://github.com/sheredom/ubench.h) instead of utest.h.
Each benchmark is a source file ending with `_bench.c`, and they're automatically linked by the Makefile, just like the main clox interpreter and the test binaries.

Unlike the tests, each benchmark file contains just one benchmark with some code that takes some time to run;
the ubench.h framework runs the workload multiple times and prints out the statistical mean runtime with outliers removed.

## Wrapping Up

I ended up with a Makefile-centered workflow for following the final part of of _Crafting Interpreters_ that could:

- Build and link C code with fully automatic dependency resolution.
- Format code on demand with `make format`, and check for unformatted code with a git pre-commit hook.
- Build and run all tests by running `make test`.
- Quickly gauge the code coverage of the tests by running `make gcovr`.
- Build all benchmarks by running `make buildbenches`.

The resulting Makefile is pretty complex, but all of the automatic dependency logic meant that if I wanted to add a new file, I could just add `foo.c`, `#include "foo.h"` and everything would Just Work. 
If I wanted to add a test, I could just write `foo_test.c` and a test binary would automatically be created, while adding a benchmark was a simple matter doing the same thing with a `foo_bench.c` file.
I don't know if I'm the first person to work out C link dependencies like this using a Makefile, but if it's been done before, I haven't seen it.

Overall, I'm pretty happy with how this all turned out.
