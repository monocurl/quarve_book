# View Providers

This lesson explains many of the core concepts of Quarve and is hence slightly long.
However, we still recommend reading (at the very least, upto and including IntoViewProvider) as it's important!

Rendered elements in Quarve are called **views**. A view has two components:
a native backing and a **view provider**. The native backing is simply a reference
to a native GUI element.
On the other hand, the view provider is responsible for controlling the
backing and appropriately filling it with content
(somewhat akin to the controller in the MVC paradigm if that helps).

Most of the time, you don't actually have to code your own view provider, but instead
reuse existing ones. Consequently, rather than implementing the ViewProvider trait,
you will more likely implement the **IntoViewProvider** trait (abbreviated IVP).

Let's take a concrete example to highlight the distinction.
Here, we have a user profile that does not know how to control a view,
however, it does not know how to convert itself into something that does.
Hence, it's an IVP.

```rust
struct Profile {
    name: String,
    color: Color,
    image: Resource,
}

impl IntoViewProvider<Env> for Profile {
    type UpContext = ();
    type DownContext = ();

    fn into_view_provider(self, env: &Env::Const, s: MSlock) -> impl ViewProvider<Env, UpContext=Self::UpContext, DownContext=Self::DownContext> {
        hstack()
            .push(
                ImageView::new(self.image)
                    .intrinsic(30, 30)
                    .padding(5)
            )
            .push(text(self.name))
            .padding(5)
            .layer(L.bg_color(self.color).radius(4))
            // notice how we use other IVPs
            // before we actually construct our view provider
            .into_view_provider(env, s)
    }
}
```

As another example, while a color (such as `BLACK`) does not know how to control a view,
it knows how to convert itself into a ViewProvider that appropriately displays
a black view. Hence, all colors are IVPs.

### IntoViewProvider
In general, you will interact with `IntoViewProvider` much more than
ViewProvider. Because it's typed so much, the default quarve project
contains a trait alias as `IVP`.

As other projects in Rust and declarative UI frameworks,
Quarve massively builds on the principle of composition through
reusable view components.

There are two principle ways to create such components. One way, which we just
saw with the user profile example is to create a struct and manually
implement the `IntoViewProvider` trait yourself.

Another way is to simply use functions. Here's a translation of the above profile
into a functional component.
```rust
fn profile(name: &str, image: Resource, color: Color) -> impl IVP {
    hstack()
        .push(
            ImageView::new(image)
                .intrinsic(30, 30)
                .padding(5)
        )
        .push(text(name))
        .padding(5)
        .layer(L.bg_color(color).radius(4))
}
```

\*Technical Note: The downside of the functional approach is that you don't have access
to the `MSlock` or `Env` (explained in later lessons).
However, in some cases you can use the `ivp_using` function if this issue ever arises,
which gives you access to the additional context (this too has some downsides
since the given lambda has to be static).

Finally, you actually specify the root IVP in the `WindowProvider::root` function.
This should be easy to find in the default project.

Note that the UpContext and DownContext associated types
of IntoViewProvider are related to ways of passing information between
and from the superview.
Don't worry about them too much, they are explained more in the next section.

## ViewProvider
Most of the time, you never have to actually implement a full view provider yourself,
but sometimes you may have to implement parts of it (i.e. delegate rest of the methods
to an existing view provider). It's also good to know the general layout model.

The ViewProvider has a few main roles (there are other minor things it does,
such as modifying environment).
1. To create and properly manage the native backing
2. To add subviews and properly lay them out
3. To respond to system and lifecycle events
4. To give proper sizing hints to the parent view so that it can layout this view

### Init Backing
The init backing function takes the following signature. It will be called once per
view_provider. Views can be removed and added from the view tree. In this case,
note that init_backing will not be called on the subsequent mounts
(so do not use it in that way, there are other ways for this).

```rust
fn init_backing(&mut self, invalidator: WeakInvalidator<E>, subtree: &mut Subtree<E>, backing_source: Option<(NativeView, Self)>, env: &mut EnvRef<E>, s: MSlock) -> NativeView;
```
This may seem quite complex, but we'll break down the individual pieces:
1. `invalidator` gives you a weak (in the Arc sense) reference to an invalidator. The invalidator
is used to mark the state of this view as dirty, thereby causing a relayout.
2. `subtree` gives you a handle by which you can add or remove subviews (among other roles)
3. `backing_source` is something more unique to Quarve. The idea is that sometimes we are creating
a view provider to replace an existing view, and we can therefore steal the replaced view's
resources rather than having to recreate them.
(similar in premise to a RecyclerView)
4. `env` is not super important for now. The environment is explained more in the [Environment](./environment) lesson.
5. `s` the state lock is also explained more in the next lesson,
but for now just think about it as a marker showing we're on the main thread

Here's an example of an `init_backing` of a LayerView. It uses some things
we haven't talked about, but hopefully the general flow makes sense.
```rust
fn init_backing(&mut self, invalidator: WeakInvalidator<E>, subtree: &mut Subtree<E>, backing_source: Option<(NativeView, Self)>, env: &mut EnvRef<E>, s: MSlock) -> NativeView {
    // whenever any of the properties of the layer
    // have changed, invalidate the state of this view
    self.layer.opacity.add_invalidator(&invalidator, s);
    self.layer.border_width.add_invalidator(&invalidator, s);
    self.layer.border_color.add_invalidator(&invalidator, s);
    self.layer.corner_radius.add_invalidator(&invalidator, s);
    self.layer.background_color.add_invalidator(&invalidator, s);

    if let Some((nv, layer)) = backing_source {
        // we are replacing an old view provider
        // so we should reuse the native view

        // it's extremely important
        // that we call this function
        // which basically recursively asks our subview
        // to reuse the backing of the old view
        // if we don't call it, the entire subtree will
        // create a brand new backing for no reason
        self.view.take_backing(layer.view, env, s);
        subtree.push_subview(&self.view, env, s);

        self.backing = nv.backing();
        nv
    }
    else {
        // this is you how you add subviews
        subtree.push_subview(&self.view, env, s);

        let nv = NativeView::layer_view(s);
        self.backing = nv.backing();
        nv
    }
}
```

### Layout Model
Quarve follows a unique layout model that is meant to be fast yet
still able to achieve complex paradigms. The idea is that there are two passes:
1. In the up phase, view providers calculate their 'preferred' sizes in a bottom-up manner.
In addition, view providers update their native backing to reflect any
state changes that may have occured since the last layout.
2. In the down phase, sizes are finalized in a top-down manner.

### Up-Phase
As mentioned, the entire job of the up-phase is to calculate the *preferred*
size of this view and any other context the superview may need to layout the current view.
Also, this is another place that you can add or remove subviews.

Here is the signature of the layout up method. Note that it returns a boolean.
This is because whenever a view provider finishes its up phase, sometimes
this view has changed its preferred size. In this case, we
Concretely, if this view provider returns true, Quarve
effectively invalidates the super view and asks it to perform its up phase
as well. If we return false, quarve assumes that there are no changes in preferred sizes
so it will not ask the superview to layout up (unless it was invalidated for other reasons)..
```rust
fn layout_up(&mut self, subtree: &mut Subtree<E>, env: &mut EnvRef<E>, s: MSlock) -> bool;
```

Here's how the LayerView layout up function looks like.
```rust
fn layout_up(&mut self, _subtree: &mut Subtree<E>, _env: &mut EnvRef<E>, s: MSlock) -> bool {
  // updates the native backing
  // In this case, we don't
  native::view::layer::update_layer_view(
      self.backing,
      self.layer.background_color.inner(s),
      self.layer.border_color.inner(s),
      self.layer.corner_radius.inner(s) as f64,
      self.layer.border_width.inner(s) as f64,
      self.layer.opacity.inner(s),
      s
  );

  // generally only called if subview propagated to here
  true
}
```

The preferred sizes we are talking about are `intrinsic`, `xsquished`,
`xstretched`, `ysquished`, and `ystretched` (if you are lazy, you can set
all of the sizes to the same value).
The expectation is that after
a view provider has its layout up called, all of these values are fully prepared
and a superview can treat them as representing the current state.
In particular, you must return the appropriate value in each of these functions:
```rust
fn intrinsic_size(&mut self, s: MSlock) -> Size;
fn xsquished_size(&mut self, s: MSlock) -> Size;
fn xstretched_size(&mut self, s: MSlock) -> Size;
fn ysquished_size(&mut self, s: MSlock) -> Size;
fn ystretched_size(&mut self, s: MSlock) -> Size;
```

### LayoutDown
In the down phase, we are guaranteed that all of the subviews have their
preferred sizes finalized. Also, we are given a suggested size by the superview.
Using all of this information, we can **finalize** a size for ourselves,
and **suggest** a size and location for each of our subviews. It's possible
that subviews may somewhat alter our suggestion, but generally when this happens
the subview only uses a portion of the suggested rect.

**Important:** In the layout down method, the view provider must then call
layout down method of all of its subviews. Otherwise, you may observe weird behavior.

Here's an example layout down method for an `HSplit` view, which adds
a splitter between two views so you can resize it horizontally. To better understand it,
here are a couple notes about geometry:
1. Quarve uses a coordinate system where (0,0) is the top left.
2. In the layout down method, you can imagine that you are given an infinitely wide
canvas. The frame parameter is a hint that your content should be included
in the (0, 0) to (frame.w, frame.h) rectangle, but is not a strict hint. The final
exclusion rectangle, the final view rectangle, and the subview rectangles can all be
thought of as being placed on this infinite canvas.
3. You can also go to negative positions, if necessary.

```rust
fn layout_down(&mut self, _subtree: &Subtree<E>, frame: Size, layout_context: &Self::DownContext, env: &mut EnvRef<E>, s: MSlock) -> (Rect, Rect) {
    // use subview information to determine
    // the min and max range of the splitter
    let (min, max) = if self.is_horizontal {
        (self.lead.xsquished_size(s).w, self.lead.xstretched_size(s).w)
    } else {
        (self.lead.ysquished_size(s).w, self.lead.ystretched_size(s).w)
    };
    // the position of the splitter
    let pos = (self.splitter.up_context(s) - BAR_WIDTH / 2.0)
        .clamp(min, max);

    if self.is_horizontal {
        // position the left and right subviews
        // as well as the splitter

        // methods such as these return the actual
        // used rectangle. We're not using it here
        self.lead.layout_down_with_context(
            Rect::new(0.0, 0.0, pos, frame.h),
            layout_context, env, s);

        // see below for what context means
        // feel free to ignore as this is isn't even
        // a great use case of it
        let ctx = pos + BAR_WIDTH / 2.0;
        self.splitter.layout_down_with_context(
            Rect::new(pos + BAR_WIDTH / 2.0 - self.splitter_size / 2.0, 0.0, self.splitter_size, frame.h),
            &ctx,
            env, s
        );
        self.trail.layout_down_with_context(
            Rect::new(pos + BAR_WIDTH, 0.0, frame.w - pos - BAR_WIDTH, frame.h),
            layout_context, env, s);
    }
    else { /* vertical case is similar and omitted */ }

    // the first value is the used rectangle of just our view
    // which is used by quarve to set the size and position of native backing
    // the second value is the "excluded" area that the
    // superview should treat this subtree as taking up
    // (so that it does not allot it to other siblings)
    // the superview only sees the second value
    // (in its own coordinate system)

    // in this case we use the entire suggestion
    (frame.full_rect(), frame.full_rect())
}
```
There are many other features of layout down that are not included in this example.
This includes using the return value of layout down of the children and
retranslating them. Nevertheless, this example is more typical of the complexity
you are likely to see.

### Context (Advanced)
Sometimes, the given size information is not enough for a superview
to make a decision about how to layout its subviews
(as is the case with flex layouts for example).
For this, we have UpContext. It is an associated type of a ViewProvider that specifies
the type of information we're going to provide to the superview.
As with the sizes, the expectation is that
after layout_up has been called, the `up_context` method returns an up-to-date
result.
In the vast majority of cases, you can set the UpContext to `()`.
```rust
fn up_context(&mut self, s: MSlock) -> Self::UpContext;
```

On the other hand, sometimes the suggested frame is not enough to make a decision
about how to layout our current view, and we would like more information from our parent.
In this case, we can set the DownContext type to the required context we desire.
Again in the vast majority of cases, you can set DownContext to `()`.
Do note that setting a non-trivial DownContext means that some caching
can't be done, but this is only a minor peformance hit.

Finally, note that for IVPs, their UpContext and DownContext associated types
are exactly those of the ViewProvider they convert to.
### Event Handling

Some of the methods of a view provider are related to events.

There is a collection of life cycle events. These are described in further detail
in the [Modifiers](./modifiers.md) lesson.
```rust
fn pre_show(&mut self, s: MSlock) {

}

fn post_show(&mut self, s: MSlock) {

}

fn pre_hide(&mut self, s: MSlock) {

}

fn post_hide(&mut self, s: MSlock) {

}
```

Also, you can respond to OS events. In general, here is how event dispatch works:
1. If the event is a mouse event and there is a focused view, dispatch to that view.
If that view consumes the event, stop.
2. Otherwise, if the event is a mouse event, dispatch the event to all views
that the cursor is on top of, stopping if any view consumes the event.
3. If the event is a key event and there is a focused view, dispatch to that view.
Continue dispatching even if the view says to consume the event.
4. If the event is a key event dispatch to all key listeners, except the focused view
if it was already dispatched to.
5. If we reach this branch, it's delegated back to the native system for dispatch.

In particular, you may want to register to become first responder if you care
about mouse events happening outside of your boundary (which you can do so by
return `EventResult::FocusAcquire`)

Here is a (partially incomplete) example of a button event handler.
```rust
fn handle_event(&self, e: &Event, s: MSlock) -> EventResult {
    if !e.is_mouse() {
        return EventResult::NotHandled;
    }

    let cursor = e.cursor();
    // it could be outside since we're focused
    let inside = cursor.x >= 0.0 && cursor.x < self.last_size.w &&
    cursor.y >= 0.0 && cursor.y < self.last_size.h;
    if inside != *self.is_hover.borrow(s) {
        self.is_hover.apply(SetAction::Set(inside), s);
    }

    if !inside || matches!(&e.payload, EventPayload::Mouse(MouseEvent::LeftUp, _)) {
        if *self.is_click.borrow(s) {
            self.is_click.apply(SetAction::Set(false), s);
            // end of click; relenquish focus
            return EventResult::FocusRelease;
        }
    } else if inside && matches!(&e.payload, EventPayload::Mouse(MouseEvent::LeftDown, _)) {
        if !*self.is_click.borrow(s) {
            (self.action)(s);

            self.is_click.apply(SetAction::Set(true), s);
            // acquire focus so we can receive
            // events even if moved outside
            return EventResult::FocusAcquire;
        }
    }

    return EventResult::NotHandled
}
```


I'm not 100% on keeping this system as there are a few quirks, but it
hasn't been a major issue thus far.

### Examples
There are tons of examples of implementing view providers in the standard library.
We recommend looking at the [source code](https://github.com/monocurl/quarve)
whenever needed (although, some view providers use internal-only functions).
