# Etc

Here are other concepts in Quarve that did not fit into any of the other lessons

## Modals
You can run a message box with a callback with code similar to this:
```rust
MessageBox::new(Some("Confirm Deletion"), None)
    .button(MessageBoxButton::Cancel)
    .button(MessageBoxButton::Delete)
    .run(|b, s| {
       println!("Pressed {:?}", b);
    });
```

Somewhat similarly, you can open and save files with `OpenFilePicker` and `SaveFilePicker`
```rust
pub fn open_file_ex() {
    // and similarly for SaveFilePicker
    OpenFilePicker::new()
        .content_types("png")
        .run(|path, s| {
            println!("Selected {:?}", path);
        })
}
```

## Event Loop
Run a function on the main thread, possibly in the next event loop.
```rust
// if the current thread is main, it will run instantly
// otherwise, it will run on the next event loop
run_main_maybe_sync(|_mslock| {
    println!("Maybe called synchronously or next event loop,\
              depending on calling thread");
}, s);

run_main_async(|_mslock| {
    println!("Called on next main loop");
})
```

## Window
Here is code for spawning another window besides the main window.

```rust
fn new_window(s: MSlock) {
    with_app(|app| {
        // where MainWindow is the window provider you want to use
        app.spawn_window(MainWindow, s)
    }, s);
}
```
