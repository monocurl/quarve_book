# Text

The

## Labels

The easiest way to create text is through the `text` function in the prelude
```rust
text("Rabbit")
```

You can also have a label that dynamically updates based on a signal
```rust
Text::from_signal(signal)
```

## TextField

A TextField can be styled similarly to Text. TextField connect their content changes
to a binding.
```rust
let text = Store::new("initial".into());
TextField::new(text.binding());
```

I don't like the focus system, but you can control when a TextField is focused
using a `TokenStore` (this is definitely subject to be changed in the future).

```rust
let focused = TokenStore::new(None);
let text1 = Store::new("alpha".into());
let text2 = Store::new("beta".into());

vstack()
    .push(
        TextField::new(text1.binding())
            .focused_if_eq(focused.binding(), 1)
    )
    .push(
        TextField::new(text2.binding())
            .focused_if_eq(focused.binding(), 2)
    );
```

\* Right now there is not that much control over the textfield in terms of its
appearance. We hope to possibly improve this in the future.

## Environment

You can specify different text settings using environment modifiers, such as below.
These affect `Text` and `TextField` views.
```rust
IVP
    .bold()
    .italic()
    .text_font("font_file_in_res/font")
    .text_size(size)
    .text_color(color)
    .text_backcolor(RED)
```

Note that environment modifiers are somewhat different than regular modifiers
in that they apply to all subviews rather than just the modifed IVP.
See the [environment lesson](./environment.md) to learn more.

## TextView (Advanced)
**TODO this section is currently poorly written and lacking much-needed visuals**

TextView is a heavy-duty text editor. It works by following an attribute model.

The state of a TextView is comprised of a series of pages. Each page has a series of runs
(also called blocks/paragraphs in other frameworks). Each run has a single string
of content, with no new lines. In addition, you can store attributes for each page,
run, or character. This is useful since, for example, the attributes can be used
to style the appeareance and layout of the characters. Moreover, you may simply
want to store custom attributes for your own application, irrespective of causing
a visual change.

There are two types of attributes: derived and intrinsic.
This is analogous to derived and normal stores, namely that the derived attributes
can be thought of solely as a function of the text content, but the intrinsic attributes
may vary even for the same text content. As an example, an auto markdown formatter
would only have derived attributes since the visual display is solely a function of the text.
An example of an intrinsic attribute would be something like a rich text editor
where we can artificially bold certain content without changing the text.
(this description is simplified).

Note that all TextViews layout their content vertically from the first page
to the last. Moreover, each page has its own separate native editor
(if you don't want this, simply use a single page).

**TODO talk about cursor and cursor state**

To actually create a TextView, you implement the `TextViewProvider` trait. Here,
you specify the intrinsic and derived attributes. You also specify
properties about how the TextView is displayed. For instance, you can add a background
to each page. You can also specify run decorations (most notably, attaching a line number
in the gutter of each line) quite easily. Finally, you can attach a decoration
to the cursor, which is useful for displaying an autocomplete window.

As there is a lot of machinery for text views, we omit an inline example
and instead refer you to this
[full example](https://github.com/monocurl/quarve/blob/main/examples/textview/src/main.rs).

### Limitations
Currently TextView only supports the same font and font size for the entirety
of the text. We only support content modification to be done on the main thread,
(attributes can be changed in any thread).

While in theory TextView should be extremely efficient,
I need to still do some optimizations to improve performance for certain scenarios.
Files <= 1000 lines long should be fine, though.
