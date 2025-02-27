---
title: Basic Counter App
description: This tutorial covers a basic counter application with Ratatui
---

A full copy of the code for this page is available in the github repository for the website at:

<https://github.com/ratatui-org/website/tree/main/code/counter-app-basic>.

## Create a new project

Create a new rust project and open it in your editor

```shell title="create counter app project"
cargo new ratatui-counter-app
cd ratatui-counter-app
$EDITOR .
```

Add the Ratatui and Crossterm crates (See [backends] for more info on why we use Crossterm).

```shell title="add dependencies"
cargo add ratatui crossterm
```

The Cargo.toml will now have the following in the dependencies section:

```toml title="Cargo.toml"
{{ #include @code/counter-app-basic/Cargo.toml:dependencies }}
```

## Application Setup

### Main Imports

In `main.rs`, add the necessary imports for Ratatui and crossterm. These will be used later in this
tutorial. In the tutorials, we generally use wildcard imports to simplify the code, but you're
welcome to use explicit imports if that is your preferred style.

```rust title="src/main.rs"
{{ #include @code/counter-app-basic/src/main.rs:imports }}
```

:::caution

Some editors remove unused imports automatically, so if you run into errors about missing types,
etc. make sure these are in place.

:::

### Main Function

A common pattern found in most Ratatui apps is that they:

1. Initialize the terminal
2. Run the application in a loop until the user exits the app
3. Restore the terminal back to its original state

The `main` function sets up the terminal by calling methods in the `tui` module (defined next), and
then creates and runs the App (defined later). It defers evaluating the result of of calling
`App::run()` until after the terminal is restored to ensure that any `Error` results will be
displayed to the user after the application exits.

Fill out the main function:

```rust title="src/main.rs"
{{ #include @code/counter-app-basic/src/main.rs:main }}
```

### TUI module

The counter app will be displayed on the [alternate screen]. This is a secondary buffer that allows
the application to avoid messing up the user's current shell. The app also enables [raw mode] so
that the app can process keys immediately without having to wait for a newline and so that the keys
are not echoed to the user's screen when pressed.

Let's implement this by creating a new module named `tui` to encapsulate this functionality into an
init and a restore functions.

Add a module to `main.rs` after the imports section:

```rust title="src/main.rs"
{{ #include @code/counter-app-basic/src/main.rs:modules }}
```

Create a new file named `src/tui.rs` for the module. Add the imports, and two new functions, `init`
and `restore`:

```rust title="src/tui.rs"
{{ #include @code/counter-app-basic/src/tui.rs }}
```

There is a PR to [simplify this boilerplate code], but for now it's most convenient to write a small
helper module to handle this.

[simplify this boilerplate code]: https://github.com/ratatui-org/ratatui/pull/280

### Application State

The counter app needs to store a small amount of state, a counter and a flag to indicate that the
application should exit. The counter will be an 8-bit unsigned int, and the exit flag can be a
simple bool. Applications that have more than one main state or mode might instead use an enum to
represent this flag.

Create an `App` struct to represent your application's state:

```rust title="src/main.rs"
{{ #include @code/counter-app-basic/src/main.rs:app }}
```

Calling `App::default()` will create an `App` initialized with `counter` set to 0, and `exit` set to
`false`.

### Application Main loop

Most apps have a main loop that runs until the user chooses to exit. Each iteration of the loop
draws a single frame by calling `Terminal::draw()` and then updates the state of the app.

Create an `impl` block for the `App` with a new run method that will act as the application's main
loop:

```rust title="src/main.rs"
impl App {
{{ #include @code/counter-app-basic/src/main.rs:run }}

    fn render_frame(&self, frame: &mut Frame) {
        todo!()
    }

    fn handle_events(&mut self) -> io::Result<()> {
        todo!()
    }
}
```

## Displaying the application

### Render a Frame

Every application needs to render the state to the screen by calling `Terminal::draw()` with a
closure that accepts a `Frame`. The application will call methods on the `Frame` to render the state
of the application (and update the cursor if needed).

In the interests of composability, it can be convenient to implement the `Widget` trait on your app
type instead of directly calling methods on frame.

First, add a new `impl Widget for &App` block. The render function will create a block with a title,
instruction text on the bottom, and some borders. Render a `Paragraph` widget with the application's
state (the value of the `App`s counter field) inside the block. The block and paragraph will take up
the entire size of the widget:

```rust title="src/main.rs"
{{ #include @code/counter-app-basic/src/main.rs:impl Widget }}
```

Next, render the app as a widget:

```rust title="src/main.rs"
impl App {

    // -- snip --

{{ #include @code/counter-app-basic/src/main.rs:render_frame }}
}
```

:::note

The closure (anonymous method) passed to the `Terminal::draw()` method must render the entire UI.
You should only call the draw method once for each pass through your application's main loop.
[See the FAQ for more information.](/faq/)

:::

### Testing the UI Output

To test how how Ratatui will display the widget when `render` is called, you can render the app to a
buffer in a test.

Add the following `tests` module to `main.rs`:

```rust title="src/main.rs"
#[cfg(test)]
mod tests {
{{ #include @code/counter-app-basic/src/main.rs:render test }}
}
```

To run this test run the following in your terminal:

```shell title="run tests"
cargo test
```

You should see:

```text title="test output"
running 1 test
test tests::render ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Interactivity

The application needs to accept events that come from the user via the standard input. The only
events this application needs to worry about are key events. For information on other available
events, see the [Crossterm events module] docs. These include window resize and focus, paste and
mouse events.

In more advanced applications, events might come from the system, over the network, or from other
parts of the application.

### Handle Events

The `handle_events` method that you defined earlier is where the app will wait for and handle any
events that are provided to it from crossterm.

Update the `handle_events` method that you defined earlier:

```rust title="src/main.rs"
impl App {

    // -- snip --

{{ #include @code/counter-app-basic/src/main.rs:handle_events }}
}
```

:::note

The [`event::read`] function blocks until there is an event. If your application needs to perform
other tasks than just the UI, then it should check whether there is a pending event by calling
[`event::poll`] with a timeout that is reasonable for your application. More about this will be
covered in a future chapter.

:::

### Handle Keyboard Events

Your counter app will update the state of the `App` struct's fields based on the key that was
pressed. The keyboard event has two fields of interest to this app:

- `kind`: It's important to check that this equals `KeyEventKind::Press` as otherwise your
  application may see duplicate events (for key down, key repeat, and key up).
- `code`: the `KeyCode` representing which specific key that was pressed.

Add a `handle_key_event` method to `App`, to handle the key events.

```rust title="src/main.rs"
impl App {

    // -- snip --

{{ #include @code/counter-app-basic/src/main.rs:handle_key_event }}
}
```

Next, add some methods to handle updating the application's state. It's usually a good idea to
define these on the app rather than just in the match statement as it gives you an easy way to unit
test the application's behavior separately to the events.

```rust title="src/main.rs"
impl App {

    // -- snip --

{{ #include @code/counter-app-basic/src/main.rs:methods }}
}
```

:::caution

Normally your application should avoid panicking, but we're leaving an overflow bug in here so we
can show how to handle errors in the next section. A real app might use `saturating_sub` and
`saturating_add` to avoid panics like this.

:::

### Testing Keyboard Events

Splitting the keyboard event handling out to a separate function like this makes it easy to test the
application without having to emulate the terminal. You can write tests that pass in keyboard events
and test the effect on the application.

Add tests for `handle_key_event` in the `tests` module.

```rust title="src/main.rs"
#[cfg(test)]
mod tests {

    // -- snip --

{{ #include @code/counter-app-basic/src/main.rs:handle_key_event test }}
}
```

Run the tests.

```shell title="run tests"
cargo test
```

You should see:

```text title="test output"
running 2 tests
test tests::handle_key_event ... ok
test tests::render ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## The Finished App

Putting this altogether, you should now have the following files:

```rust collapsed title="main.rs (click to expand)"
{{#include @code/counter-app-basic/src/main.rs }}
```

```rust collapsed title="tui.rs (click to expand)"
{{#include @code/counter-app-basic/src/tui.rs }}
```

### Running the app

Make sure you save all the files and that the [imports] listed above are still at the top of the
file (some editors remove unused imports automatically).

[imports]: #main-imports

Now run the app:

```shell title="run the app"
cargo run
```

You will see the following UI:

![basic-app demo](./basic-app.gif)

Press the **Left** and **Right** arrow keys to interact with the counter. Press **Q** to quit.

Note what happens when you press **Left** when the counter is 0.

![basic-app demo](./basic-app-error.png)

On a Mac / Linux console you can run `reset` to fix the console. On a Windows console, you may need
to restart the console to clear the problem. We will properly handle this in the next section of
this tutorial on [Error Handling].

## Conclusion

By understanding the structure and components used in this simple counter application, you are set
up to explore crafting more intricate terminal-based interfaces using `ratatui`.

[Error Handling]: /tutorials/counter-app/error-handling/
[backends]: /concepts/backends/
[alternate screen]: /concepts/backends/alternate-screen/
[raw mode]: /concepts/backends/raw-mode/
[Crossterm events module]: https://docs.rs/crossterm/latest/crossterm/event/index.html
[`event::read`]: https://docs.rs/crossterm/latest/crossterm/event/fn.read.html
[`event::poll`]: https://docs.rs/crossterm/latest/crossterm/event/fn.poll.html
