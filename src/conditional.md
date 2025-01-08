# Conditional

Here we show how to show or hide views conditionally.

## If Else
The simplest conditional is the if (and else) block. These statements
conditionally hide or show views based on the given boolean signal.
Here's the syntax:
```rust
let shown = Store::new(false);
let shown_binding = shown.binding();

hstack()
    .push(button("toggle color", move |s| {
        let curr = *shown_binding.borrow(s);
        shown_binding.apply(SetAction::Set(!curr), s);
    }))
    .push(
        // dont like this syntax that much
        // but i also think a macro would be overkill
        view_if(shown.signal(), BLUE.intrinsic(50, 50))
            // else if would go here
            .view_else(RED.intrinsic(50, 50))
    )
```

## View Match
A common paradigm is to only show one view out of a set of views, depending on some
state (think about a router). This can be accomplished by the `view_match!` macro.
It's conceptually similar to a match statement based upon the value of some signal.
However, a key distinction is that the different arms are allowed to take different
types.

The general syntax is as follows
```rust
view_match!(signal;
    pattern_1 => IVP1,
    pattern_2 => IVP2,
    _ => DEFAULT_IVP
)
// sometimes you don't want to match exactly on the content of a signal
// but rather match on the content of a signal after some initial
// operation
view_match!(signal, |content| mapped_expr;
    pattern_1 => IVP1,
    pattern_2 => IVP2,
    _ => DEFAULT_IVP
)
```

Here's an example:
```rust
fn mux_demo() -> impl IVP {
    let selection = Store::new(None);
    let selection_sig = selection.signal();

    let mux = view_match!(selection_sig, |val: &Option<String>| val.as_ref().map(move |q| q.as_str());
        Some("Alpha") => text("alpha text").bold(),
        Some("Beta") => button("beta button", |_s| println!("clicked!")).italic(),
        _ => text("Please select an option")
    );

    // macro syntax of vstack
    vstack! {
        mux;
        Dropdown::new(selection.binding())
            .option("Alpha")
            .option("Beta");
    }
}
```

The included examples are available in
[this project](https://github.com/monocurl/quarve/tree/main/examples/conditional).
