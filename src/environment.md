# Environment

First, we note that most projects can get away with using the standard environment
entirely and that modification is only necessary in some niche scenarios.

Nevertheless, environments are another method for passing information.
They come in two flavors: the constant environment and variable environment.

As a comment, for a given window all IVPs and VPs must be implemented with respect
to the window's environment. Theoretically separate windows can have separate environment
types, but this is uncommon.

## Constant Environment
The constant environment is context that is passed to the entire view hierarchy
of a window. This can be used to pass references to bindings, menu channels, portals,
color schemes, and anything else you find relevant. Note that the value of the
constant environment is specified in the `root_environment()` function of your
`Environment` implementation.

The main way the constant environment is used is in the `into_view_provider` function.
Namely, whenever a struct is converted into a view provider, it can use the constant
environment freely. This way, you do not have to have the caller of the IVP explicitly
provide you with e.g. the MenuChannel, but can instead query it during conversion time.

A final note is that despite being called the constant environment, it's theoretically
possible to change it over time (but it will still be observed the same by all views
for any given timestamp).
However, this is recommended against and may cause logical errors.

Here's a (snipped) section of Quarve's library code where we take advantage
of the constant environment to gain access to the appropriate channels that we need.
In particular, for each page of a TextView, we need to be able to respond to events
such as select all.
```rust
fn into_view_provider(self, env: &E::Const, _s: MSlock) -> impl ViewProvider<...> {
    // use the standard env to find the appropriate menu channel
    // that a page needs to respond to
    let env = env.as_ref();
    PageVP {
        /- snip -/
        select_all_menu: env.channels.select_all_menu.clone(),
        cut_menu: env.channels.cut_menu.clone(),
        copy_menu: env.channels.copy_menu.clone(),
        paste_menu: env.channels.paste_menu.clone(),
    }
}
```

## Variable Envrionment
While the constant environment is uniform throughout the view tree,
the variable environment can be changed for certain subtrees. You can think about it
as each view in the tree has its own copy of the variable environment personalized
to itself (of course, this is not how it is actually implemented for efficiency reasons).

This is useful because it allows you to configure settings for only portions
of the view hierarchy. For instance, the `.text_size` modifier is actually an environment
modifier that sets a special variable in the environment to the specified size.
Then, text related views in this subtree read this value from the environment
and consequently render their text appropriately. This highlights how variable
environment modifiers allow you to give information to many views at once,
which is much nicer than manually specifying the text size for every single
label, in this example.

```rust
fn var_env_example() -> impl IVP {
    vstack()
        .push(text("rabbit"))
        .push(text("bunny"))
        .push(
            text("strawberry")
                // explicitly override the parent environment modifier
                .text_size(20)
        )
        // set the text size for this entire subtree
        // note that vstack is not even text related,
        // but we can still apply the modifier here!
        .text_size(14)
}
```


**Advanced:** You can write your own environment modifier by implementing the
`EnvironmentModifier` trait. These work by having methods to apply the changes
of this modifier to the environment, and a corresponding one to undo the changes
(thereby restoring the environment to its original state). To actually use the modifier,
call `.env_modifier(<your modifier>)` on the target IVP. Of course, feel free to
create a convenience modifier to simplify the syntax.
