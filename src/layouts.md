# Layouts

## Types of Layouts

The common types of layouts are `vstack`, `hstack`, `zstack`, and
`flexstack`. The first three are more or less straightforward, but `flexstack`
takes from

## Heterogenous layouts

For each of these, you can layout a heterogenous collection of elements.
```rust
// vstack() is an alias for VStack::hetero()
VStack::hetero()
    .push(first_view)
    .push(second_view)
    ...
```

## Iterator
If you have a collection of items, and for each one you want to display
a view based off of that item,
you can use the `vmap`, `hmap`, `zmap`, and `flexmap` functions.

Here's a toy example.
```rust
let strings = ["The", "rabbit", "jumped", "over", "the", "moon"];
strings
    .vmap(|string| text(string));
```

## Binding
If you have a binding of a vector of data, you can have a layout
of the mapped views, which updates automatically as the binding updates.

```rust
let strings: impl Binding<Filterless<Vec<String>>>;
strings
    .binding_vmap(|string| text(string));
```

## Signal
Finally, there's the analogous method for signals of vectors. If you can,
try to use `binding_vmap` or its analogs instead as `sig_vmap` is significantly
slower.

```rust
let strings: impl Signal<Target=Vec<String>>;
strings
    .sig_vmap(|string| text(string));
```

## Using Options
Sometimes you want to customize a vstack to adjust the spacing, for example.
This can be done using the options for that layout.
```rust
VStack::hetero_options(
    VStackOptions::default()
        .direction(VerticalDirection::Up)
        .align(HorizontalAlignment::Trailing)
        .spacing(0.0)
)
    .push(...)
// signal
signal.sig_vmap_options(|elem| ...,
    VStackOptions::default()
        .spacing(0.0)
)
// iterator
into_iteratoe.vmap_options(|elem| ...,
    VStackOptions::default()
        .spacing(0.0)
)
// binding
binding.binding_vmap_options(|elem| ...,
    VStackOptions::default()
        .spacing(0.0)
)
```

## FlexStack
The behavior of `vstack`, `hstack`, and `zstack` are more or less intuitive,
but `flexstack` is slightly more complex. It is based off html flexbox
and *most* features present in the html edition are present here as well.
In the options of the parent view, you can specify things such
as the flex direction, whether it wraps, etc.
For each subview, you can use the `flex` modifier to set options about how this
specific child should be sized.
```rust
FlexStack::hetero_options(FlexStackOptions::default().gap(0.0))
    .push(
        text("first")
            .frame(F.intrinsic(200, 30).squished(40, 30).unlimited_stretch())
            .border(RED, 1)
            .flex(FlexContext::default().grow(1.0))
    )
    .push(
        text("second block of text")
            .frame(F.intrinsic(100, 30).unlimited_stretch())
            .border(BLUE, 1)
            .flex(FlexContext::default().grow(3.0))
    )
    .push(
        text("final")
            .frame(F.intrinsic(300, 30).unlimited_stretch())
            .border(BLACK, 1)
            .flex(FlexContext::default().grow(0.5))
    );
```

See more options in use in the
[flex](https://github.com/monocurl/quarve/tree/main/examples/flex) example.

Remark: one of the nice things about Quarve is that
the layout system is general enough that `flexstack` doesn't
have to be axiomatized,
i.e. flexstack doesn't use any internal-only APIs
so a user using the quarve library could fully recreate flexstack and theoretically
more complex generalizations.

## HSplit
You can also easily create a vertical/horizontal split view.
It's generally recommended to put each of the left and right
into their own frames so that you can be explicit about the minimum
and maximum content size.
```rust
HSplit::new(left, right)
```

## LayoutProvider
What if you have a set of views and you need to layout the children in
some complex manner. One option is to manually implement view provider
and handle all details of the layout process. However, a slightly easier way
is to implement `LayoutProvider` instead.
Given any struct that implements layout provider, you can convert it into
a view provider by using the `.into_layout_view_provider` method.

This handles the native backing automatically. Apart from that, the
methods and layout process is more or less the same as with vanilla `ViewProvider`
(revisit that lesson if needed).

Note that another common use case of `LayoutProvider` is to handle mouse events,
which is otherwise difficult to do.

In fact, `HSplit` and `VSplit` use a LayoutProvider
[under the hood](https://github.com/monocurl/quarve/blob/main/quarve/src/view/layout.rs).

## VecLayoutProvider (Advanced)
Note that layout providers such as `vstack`, `hstack`, `zstack`, `flexstack`, etc
can be defined on any number of subviews.
For these, it would be nice to have a simple interface for performing the layout
and automatically creating the relevant `hetero`, `vmap`, `sig_vmap`, etc methods.

Indeed, you can implement the `VecLayoutProvider` yourself to specify how to layout
an arbitrary number of subviews. To get the relevant methods, quarve provides
a set of macros, such as `impl_hetero_layout` to make it quite simple.

See how vstack and others are implemented
[here](https://github.com/monocurl/quarve/blob/main/quarve/src/view/layout.rs)
(you may have to scroll quite far, it's in the `vec_layout::vstack` module)
