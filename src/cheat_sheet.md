# Cheat Sheet

Again, views are the heart of Quarve. There is a decent number of provided views,
of which we will cover commonly used ones.
(really, the standard library has a wide collection of ViewProviders and
IntoViewProviders rather than Views, but we'll generally blur the distinction for sake
of notational convenience).

Here is a guide
Some concepts are mentioned in more detail in later lessons. Treat this lesson more as a
'cheat sheet' of good to know views rather than full explanation.

## General
Construct basic text with the `text` function located in the prelude.
```rust
text("rabbit")
```

All colors are IVPs so
(usually you will also want to set a size, hence the second line)

## Controls

Dropdown Menu

TextField

Button

\*There are many controls that one would expect from a UI library that are yet to
be added. I apologize for this and may add them in the future if there is interest.

## Layouts

You can use a `vstack`, `hstack`, and `zstack` to c
vstack()

These and other layout views are covered in more detail [here](./layouts.md).
