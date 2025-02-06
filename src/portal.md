# Portals

To provide some motivation of portals, we note the following two problems with
declarative UI libraries (and partially UI libraries in general).
1. Typically, the state tree more or less aligns with the view hierarchy,
but sometimes there are mismatches.
Concretely, suppose a view depends on state so that it makes sense
to initialize it deep in the view hierarchy, but layout-wise we want it in the toolbar
(which is completely separate in the view hierarchy). This will usually lead
to ad-hoc solutions, such as making the state globally accessible.
2. Declarative libraries are not great for moving a view around in a view hierarchy.
In particular, suppose that in certain cases, we want a view to be positioned in the
left hand side of the screen as a popover, but upon clicking a button, it should be displayed in the
middle as the main content.
This may be solved by having conditional views at each possible location,
but that strategy is suboptimal since it does not properly
transfer state between locations and is also memory inefficient.

## The solution: Portals

How do we provide such an interface in a clean, elegant abstraction?
The answer is inspired by Rust's common paradigm of senders and receivers.
In particular, the idea is that in one location of the view hierarchy,
we will have a portal sender. The sender's job is to provide the portal with created
content. In another location, we have the portal receiver. This acts as the "mounting point"
for where the view will actually be displayed. Note that the view behaves
*exactly* as if it was placed in the spot of the receiver (i.e. it inherits the
receiver's environment, not the sender's).

This solves the first problem since the place where we create the view
can now be completely separated from the place where it needs to be mounted.

It also solves the second problem since if we create a new portal receiver
(on the same portal) at a new location, the content will be moved towards there.

## Example
Here's a (toy) example where we show a way of fixing the first problem.
```rust
pub fn basic_portal() -> impl IVP {
    // create the communication channel
    let p = Portal::new();

    vstack()
        .push(
            // mount the contents on this portal
            // in this example the sender and receiver
            // are in the same function so there's little benefit
            // but in theory they can be very far apart in the view tree
            PortalReceiver::new(&p)
        )
        .push(
            RED.intrinsic(100, 100)
        )
        .push(
            // send a blue view as the content
            // Note that this sender is active only
            // whenever this view is shown
            // not important in this example
            // but sometimes it can be useful
            GREEN
                .intrinsic(100, 100)
                .portal_send(&p, BLUE.intrinsic(100, 100))
        )
}
```

Here's another toy example where we show how the second problem can be fixed.
```rust
pub fn dynamic_portal(s: MSlock) -> impl IVP {
    let p = Portal::new();

    let counter = Store::new(0);
    // imagine more complex conditions for yourself
    let left = counter.map(|c| *c % 2 == 1, s);
    let right = counter.map(|c| *c % 2 == 0, s);

    let text = Store::new("".to_string());

    hstack()
        .push(
            view_if(left, PortalReceiver::new(&p))
                .view_else(BLACK.intrinsic(100, 30))
        )
        .push(
            button("switch", move |s| {
                counter.apply(NumericAction::Incr(1), s)
            })
        )
        .push(
            view_if(right, PortalReceiver::new(&p))
                .view_else(BLACK.intrinsic(100, 30))
        )
        .push(
            // empty view is nice for portal sending
            // since it will never have children
            EmptyView
                .portal_send(&p, TextField::new(text.binding())
                    .border(RED, 1)
                    .intrinsic(100, 30)
                )
        )
}
```

Note that you can only have the portal sender be mounted on an view
that has no children. Otherwise, we run the risk of infinite loops while performing
layouts.

## Final Remarks
Although these are both toy examples, it should hopefully paint the picture
of how portals can be powerful. However, we do note that excessive use of portals
can cause long layout times and lead to stutter, so use them only if necessary.
Specifically, avoid portals within portals within portals. It's (somewhat) okay
if the different usages of portals are parallel/sibilings,
but nested portals can be especially bad when needing to rerender.
