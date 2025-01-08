# Modifiers

Modifiers are methods that transform one IVP into another IVP by adding certain
functionalities. They are pretty intuitive and allow for complex user interfaces.

We emphasize that the order of modifiers is important. In some cases,
code won't even compile if you switch the order.

## Positional Modifiers

Apply padding to an ivp
```rust
ivp
    .padding(amount)

// dynamically based on a signal
ivp
    .padding_signal(signal)

// just a single edge
ivp
    .padding_edge(amount, quarve::geo::edge::LEFT | quarve::geo::edge::RIGHT /* or likewise*/)
```

Offset the ivp by some amount
```rust
ivp
    .offset(dx, dy)
// based on a signal
ivp
    .offset_signal(dx_signal, dy_signal)
```

In quarve's positioning model, a view has 5 associated sizes:
intrinsic, xsquished, xstretched, ysquished, ystretched. Layouts such as
`vstack` use this information to appropriately position their subviews.

Sometimes, you may want to manually specify some of these sizing hints, which is
where the `frame` modifier comes in. The formal behavior is a bit hard to describe
, but the essense is that you are "placing" the target IVP into the
given frame. If it doesn't fill up the entir content,
you can also specify which alignment to use. (this is inspired by SwiftUI frame).

```rust
ivp
    .frame(F
        .align(Alignment::Leading)
        // set both x squished and y squished
        // note that by default, the squished and stretched size is the same
        // as the intrinsic size
        .squished(100, 100)
        .intrinsic(200, 200)
        .stretched(300, 300)
    )

ivp
    .intrinsic(width, height)
// shorthand for .frame(F.intrinsic(width, height))
```

## Visual Modifiers
You can alter many visual properties about a view using the `layer` modifier

```rust
ivp
    .layer(L
        .bg_color(BLUE)
        .border(RED, 1)
        // corner radius
        .radius(4)
        // for all of these, you can also use signals
        .radius_signal(sig)
    )
// shorthand for border
ivp
    .border(RED, 1)
// shorthand for bg_color
ivp
    .bg_color(RED)
```

Set a cursor for a view
```rust
ivp
    .cursor(Cursor::Pointer)
```

## Conditional Modifiers
Sometimes, you may only want to apply modifiers conditionally.
For many modifiers of the standard library (but not all), this can be done.

In particular, we can use the `when` (meta-) modifier that only
applies a set of modifiers conditionally.

```rust
IVP
    .when(cond, |ivp | { ivp
        // positional modifiers
        .offset(100, 100)
        .padding(100)
        .frame(F.intrinsic(100, 100))
        // env modifiers
        .text_color(RED)
        .bold()
        // layer
        .layer(L.bg_color(RED))
        // foreground or background
        .background(RED.layer(L.radius(4)))
        // portals
        .portal_send(&p, BLUE)

        // however, life cycle methods
        // such as .pre_show are not supported
    })
```

## Lifecycle Modifiers

You can get notifications for when a view is mounted or unmounted onto the Window.
Do note that the same view can be mounted and unmounted multiple times.

```rust
ivp
    .pre_show(|s| { ... }) // called before children and before being shown
    .post_show(|s| { ... }) // called after children and after being shown
    .pre_hide(|s| { ... }) // called before children and before being hidden
    .post_hide(|s| { ... }) // called after children and after being hidden
```

## Environment Modifiers
Environment modifiers change properties for the entire subtree
of an ivp. The most common ones are related to text.
```rust
vstack()
    .push(text("rabbit"))
    .push(
        text("bunny")
            .bold()
    )
    .push(
        text("strawberry")
            // explicitly override the parent environment modifier
            .text_size(20)
    )
    // set the text size for this entire subtree
    // note that vstack is not even text related,
    // but we can still apply the modifier here!
    .text_size(14)
    .italic()
```

See [Environment](./environment.md) and [Text](./text.md) lessons for more.

## Key Listener

You can see whenever keys are pressed via the key listener method,
which also tells you the currently active modifiers.

```rust
ivp
    .key_listener(|keys, modifiers, s| {
        println!("Pressed {:?}", keys)
    })
```

## Creating your own Modifiers (Advanced)
Rather than fully implementing your own view provider from scratch,
you can create a view provider that "wraps" another view provider and delegate
most of the view provider methods to the source view provider (composition).
Then, for whatever functionality you want to add, you should properly modify
the necessary methods.
This is one of the advantage of the view provider model:
we can append functionality to a view without allocating more native backings
(which are really expensive).

Be careful, as if you don't properly call the source view provider method properly,
the observed behavior may be unexpected. Also, in addition to creating a wrapping view provider, you typically create an
associated wrapping IVP as well.

Example: [ShowHide](https://github.com/monocurl/quarve/blob/main/quarve/src/view/modifers.rs)
modifier (you may have to scroll to `ShowHide`)

### Provider Modifier
While this approach works, it to instead recommended to implement
the `ProviderModifier`. This is less versatile than the
composite approach, but it allows the modifier to be used in `when` blocks
freely.

To see a full example of this, take a look at the implementation of the
[Offset](https://github.com/monocurl/quarve/blob/main/quarve/src/view/modifers.rs) modifier
(you may have to scroll down to find `OffsetModifiable` and related structs).
