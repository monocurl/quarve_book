# Cheat Sheet

Some concepts are mentioned in more detail in later lessons. Treat this lesson more as a
'cheat sheet' of good to know methods rather than a full explanation.

## Views
Construct basic text with the `text` function located in the prelude.
```rust
text("rabbit")
```

Display image
```rust
ImageView::new("path_to_resource_in_res_folder")
```

All colors are IVPs (usually you want to set a size, hence the second line)
```rust
BLACK
    .intrinsic(100, 100)
```

Dropdown Menu
```rust
let selection = Store::new(None);

Dropdown::new(selection.binding())
    .option("Alpha")
    .option("Beta");
```

TextField
```rust
let text = Store::new("initial".into());
TextField::new(text.binding());
```

Text Button
```rust
button("Label", |s| {...})
```

\*There are many controls that one would expect from a UI library that are yet to
be added. I apologize for this and may add them in the future if there is interest.

## Layouts

You can use a `vstack`, `hstack`, or `zstack` to organize content easily.
There are also flex layouts, but these are needed less.

You can either layout heterogenous data known at compile time,or dynamically
based off of bindings and signals.
```rust
// hetereogenous
vstack()
    .push(ivp1)
    .push(ivp2)

// binding
binding.binding_vmap(|content, s| text(content.to_string());

// signal (slow)
signal.sig_vmap(|content, s| text(content.to_string());
```

## Modifiers

Apply padding to an ivp
```rust
ivp
    .padding(amount)
```

Offset the ivp by some amount
```rust
ivp
    .offset(dx, dy)
```

Set the intrinsic size, essentially allocating a rectangle of space.
```rust
ivp
    .intrinsic(width, height)
// shorthand for .frame(F.intrinsic(width, height))
```

Setting text attributes
```rust
ivp
    .text_font("font_file_in_res/font")
    .text_size(size)
    .text_color(color)
```

Set the background color
```rust
ivp
    .bg_color(COLOR)
```

Add a foreground view
```rust
ivp
    .foreground(attraction)
```

Add a background view
```rust
ivp
    .background(attraction)
```

Lifecycle methods
```rust
ivp
    .pre_show(|s| { ... }) // called before children and before being shown
    .post_show(|s| { ... }) // called after children and after being shown
    .pre_hide(|s| { ... }) // called before children and before being hidden
    .post_hide(|s| { ... }) // called after children and after being hidden
```

## Conditional

if else
```rust
view_if(condition_signal, IVP1)
    .view_else(IVP2)
```

view match
```rust
view_match!(signal;
    0 => arm1ivp,
    1 => arm2ivp,
    _ => default_arm_ivp
);
```
