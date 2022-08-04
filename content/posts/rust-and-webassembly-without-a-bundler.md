+++
date = 2022-08-04
title = "Rust and WebAssembly without a Bundler"
taxonomies.tags = ["javascript", "programming", "rust", "webassembly", "webdev"]
+++

If you're just getting into compiling your Rust code into WebAssembly and want to load it in a web browser, you might be taken aback by the multitude of ways of doing so.
This seems to be due to the differing pace of web browsers implementing web platform features over the years.
A lot of entry-level guides to using Rust and WebAssembly make use of a JavaScript bundler for convenience, but this obscures the relationship between Rust, WebAssembly, JavaScript and HTML, so instead we're going to try doing this all by hand.
Specifically, we're going to compile some Rust code into WebAssembly and do a run-down of the ways to load it directly in a web page using just JavaScript.
If you want to follow along at home, make sure you have Rust installed and the `wasm32-unknown-unknown` target:

```
rustup target add wasm32-unknown-unknown
```

We're going to look at these loading methods through the perspective of compatibility with three desktop web browsers: Chrome, Firefox and Safari.
I'll be consulting the extremely-helpful [Can I use](https://caniuse.com) website for this info.

Ready?
Okay, let's go!

<!-- more -->

## WebAssembly

<https://caniuse.com/wasm>

- Chrome 57: March 9, 2017
- Firefox 52: March 7, 2017
- Safari 11: September 19, 2017

There's no point trying to load WebAssembly in a browser if it doesn't support it.
However, if it does, we can generally assume the presence of other things like [arrow functions](https://caniuse.com/arrow-functions), which is covenient.

## `WebAssembly.instantiate`

This method uses [`WebAssembly.instantiate`](https://caniuse.com/mdn-javascript_builtins_webassembly_instantiate), which is supported by all versions of Chrome, Firefox and Safari that also support WebAssembly.

We'll create a new Rust project that gets JavaScript to call a Rust function that in turn calls a JavaScript function, then have JavaScript call another Rust function:

```
cargo new wasm-instantiate
```

Add files with the following contents:

{% code_file(filename="wasm-instantiate/src/main.rs") %}
```rust
extern "C" {
    pub fn log_number(number: u32);
}

fn main() {
    unsafe {
        log_number(42);
    }
}

#[no_mangle]
pub extern "C" fn add(a: u32, b: u32) -> u32 {
    a + b
}
```
{% end %}

{% code_file(filename="wasm-instantiate/index.html") %}
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Rust WASM Demo</title>
  </head>
  <body>
    <script>
      let imports = {
        env: {
          log_number: (number) => console.log(`Number from Rust: ${number}`)
        }
      };
      fetch('target/wasm32-unknown-unknown/release/wasm-instantiate.wasm')
        .then((response) => response.arrayBuffer())
        .then((bytes) => WebAssembly.instantiate(bytes, imports))
        .then((result) => {
          result.instance.exports.main();
          const sum = result.instance.exports.add(1, 2);
          console.log(`1 + 2 = ${sum}`);
        });
    </script>
  </body>
</html>
```
{% end %}

Build the WebAssembly blob from the Rust code:

```
cargo build --release --target=wasm32-unknown-unknown
```

Fire up a local web server (in Ubuntu Linux using `python3 -m http.server` works) and visit the `index.html` page in your browser.
You should see the following in your JavaScript console:

```
Number from Rust: 42
1 + 2 = 3
```

What happened here is that the WebAssembly blob was fetched by the browser before being turned into an [`ArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer) and fed into `WebAssembly.instantiate`.
The `instance` field of the result can then be used to call the Rust-defined `main` and `add` functions.{% note(num=true) %}Not sure why `main` can get away without `#[no_mangle]` and `pub extern "C"` when `add` needs it.{% end %}

The biggest benefit of using `WebAssembly.instantiate` is that it works in pretty much every browser that supports WebAssembly itself.
However, there's a downside:

> **Warning:** This method is not the most efficient way of fetching and instantiating wasm modules.
> If at all possible, you should use the newer [`WebAssembly.instantiateStreaming()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/instantiateStreaming) method instead,
> which fetches, compiles, and instantiates a module all in one step, directly from the raw bytecode, so doesn't require conversion to an [`ArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer).
>
> <footer><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/instantiate">WebAssembly.instantiate() - JavaScript | MDN</a></footer>

Which brings us to our next WebAssembly loading method...

## `WebAssembly.instantiateStreaming`

<https://caniuse.com/mdn-javascript_builtins_webassembly_instantiatestreaming>

- Chrome 61: September 5, 2017
- Firefox 58: January 23, 2018
- Safari 15: September 20, 2021 (!!!)

This approach is almost the same as using `WebAssembly.instantiate`, but the `fetch` block in JavaScript looks like this instead:

```javascript
fetch('target/wasm32-unknown-unknown/release/wasm-instantiate.wasm')
  .then((response) => WebAssembly.instantiateStreaming(response, imports))
  .then((result) => {
    result.instance.exports.main();
    const sum = result.instance.exports.add(1, 2);
    console.log(`1 + 2 = ${sum}`);
  });
```

That is, `WebAssembly.instantiateStreaming` replaces the `arrayBuffer` and `WebAssembly.instantiate` calls.
The benefits of this are described in the quote in the previous section.

Safari took a whopping four years to support this over plain `WebAssembly.instantiate`!

## `wasm-bindgen --target no-modules`

This method uses [async functions](https://caniuse.com/async-functions), which are supported in all versions of Chrome, Firefox and Safari that also support WebAssembly.

So far we've been compiling and using raw WebAssembly from JavaScript.
As you may have noticed from the, uh, unambitious code samples, JavaScript can communicate with WebAssembly with numbers, peer into it's memory, and... not do much else.
It'd be nice to be able to send bigger things like strings back and forth between JavaScript and WebAssembly.

Enter [`wasm-bindgen`](https://rustwasm.github.io/docs/wasm-bindgen/).
It allows us to write functions that accept things like strings and objects in Rust and call them directly on the JavaScript side.
Specifically, we'll want the `wasm-bindgen` command line utility:

```
cargo install wasm-bindgen-cli
```

Now start with a new Rust project; we'll be cribbing from [the `wasm-bindgen` Guide](https://rustwasm.github.io/docs/wasm-bindgen/examples/without-a-bundler.html):

```
cargo new --lib no-modules
```

{% code_file(filename="no-modules/Cargo.toml") %}
```toml
[package]
name = "no-modules"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2.82"

[dependencies.web-sys]
version = "0.3.59"
features = [
  'Document',
  'Element',
  'HtmlElement',
  'Node',
  'Window',
]
```
{% end %}

{% code_file(filename="no-modules/src/lib.rs") %}
```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen(start)]
pub fn main() -> Result<(), JsValue> {
    let window = web_sys::window().expect("window");
    let document = window.document().expect("document in window");
    let body = document.body().expect("body in document");

    let val = document.create_element("p")?;
    val.set_inner_html("Hello from Rust!");

    body.append_child(&val)?;

    Ok(())
}

#[wasm_bindgen]
pub fn add(a: u32, b: u32) -> u32 {
    a + b
}
```
{% end %}

Compile it using `cargo`:

```
cargo build --release --target=wasm32-unknown-unknown
```

The resulting WebAssembly blob at `target/wasm32-unknown-unknown/release/no_modules.wasm`{% note(num=true) %}Not sure why `cargo` converted the dash in the package name into an underscore here, but not in the previous methods.{% end %} needs be run through the `wasm-bindgen` tool:

```
wasm-bindgen target/wasm32-unknown-unknown/release/no_modules.wasm \
    --out-dir .                                                    \
    --target no-modules                                            \
    --no-typescript
```

This should produce a `no_modules.js` file that our web page will load, as well as a `no_modules_bg.wasm` file that `no_modules.js` will load in turn.
Next, the web page itself.

{% code_file(filename="no-modules/index.html") %}
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Rust WASM Demo</title>
  </head>
  <body>
    <script src="no_modules.js"></script>
    <script>
      const { add } = wasm_bindgen;

      async function run() {
        await wasm_bindgen('./no_modules_bg.wasm');
        const sum = add(1, 2);
        console.log(`1 + 2 = ${sum}`);
      }

      run();
    </script>
  </body>
</html>
```
{% end %}

Loading this should show "Hello from Rust!" in the browser, and the following in the JavaScript console:

```
1 + 2 = 3
```

Loading the `no_modules.js` file in a `<script>` tag creates a global `wasm_bindgen` variable that's both a function and an object with properties.
Calling `wasm_bindgen` as a function acts like calling the `main` function in Rust, since it was annotated with `#[wasm_bindgen(start)]`.
Other functions annotated with just `#[wasm_bindgen]` like our `add` Rust function exist as properties on the `wasm_bindgen` object.

## `wasm-bindgen --target web`

<https://caniuse.com/es6-module>

- Chrome 61: September 5, 2017
- Firefox 60: May 9, 2018
- Safari 11: September 19, 2017

This form of WebAssembly loading makes use of `<script type="module">`.
As you might have guessed, this time we'll be using the `wasm-bindgen` tool to produce a *module* that the web browser will load.

Start a new Rust project:

```
cargo new --lib web
```

{% code_file(filename="web/Cargo.toml") %}
```toml
[package]
name = "web"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2.82"

[dependencies.web-sys]
version = "0.3.59"
features = [
    'Document',
    'Element',
    'HtmlElement',
    'Node',
    'Window',
]
```
{% end %}

The `web/src/lib.rs` file is identical to `no-modules/src/lib.rs` from the `wasm-bindgen --target no-modules` section, so copy it over now.

Build the Rust code the same way:

```
cargo build --release --target=wasm32-unknown-unknown
```

We'll use the `wasm-bindgen` tool again, but this time we'll pass `--target web` to create a module:

```
wasm-bindgen target/wasm32-unknown-unknown/release/web.wasm \
    --out-dir .                                             \
    --target web                                            \
    --no-typescript
```

Again, we get `web.js` and `web_bg.wasm` files.
We want to load the `web.js` in our web page.

{% code_file(filename="web/index.html") %}
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Rust WASM Demo</title>
  </head>
  <body>
    <script type="module">
      import init, { add } = from './web.js';

      async function run() {
        await init();
        const sum = add(1, 2);
        console.log(`1 + 2 = ${sum}`);
      }

      run();
    </script>
  </body>
</html>
```
{% end %}

Running a local web server and viewing this in your web browser should once again show "Hello from Rust!" and the following in the JavaScript console:

```
1 + 2 = 3
```

`web.js` is now a module, so `init` is the handle to the `#[wasm_bindgen(start)]`-annotated `main` Rust function, and the `#[wasm_bindgen]`-annotated `add` Rust function is exported as expected.

## Module with Top-Level `await`

<https://caniuse.com/mdn-javascript_operators_await_top_level>

- Chrome 89: March 1, 2021 (!!!)
- Firefox 89: June 1, 2021 (!!!)
- Safari 15: September 20, 2021 (!!!)

This is a slight tweak to the JavaScript code inside the `<script type="module">` in the web page:

```javascript
import init, { add } from './web.js';
await init();
const sum = add(1, 2);
console.log(`1 + 2 = ${sum}`);
```

This saves a bit of typing, but requires support for `await` at the JavaScript top level, which isn't supported for browsers stuck in the 3-4 year gap between 2017 and 2021.

## Conclusion

If you only need to load raw WebAssembly, `WebAssembly.instantiateStreaming` is only newer than `WebAssembly.instantiate` by months, except for Safari, which lags by years.

For the enhancements provided by `wasm-bindgen`, `--target no-modules` has maximum compatibility, and `--target web` is only newer by months.
There's a big multi-year gap for top level `await` support; use it at your own peril.
