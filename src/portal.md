# Portals
Portals are (to the best of the knowledge of the authors, which is not that much)
a unique contribution of Quarve.
To provide some motivation, we note the following two problems with
declarative UI libraries (and partially UI libraries in general).
1. Sometimes we may need.
Concretely, a view may need, but layout wise should be
2. Declarative libraries are not great for moving a view around in a view hierarchy.
What if at some time I want a view

## The solution: Portals

How do we provide such an interface in a clean, elegant abstraction? The answer is inspired
by Rust's common paradigm of senders and receivers. In particular,


## Example

Here's a (toy) example where we show a way of fixing the first problem.

Here's another toy example where we show

## Final Remarks
Although these are both toy examples, it should hopefully paint the picture
of how portals can be powerful. However, we do note that excessive use of portals
can cause long layout times, leading to stutter, so use them only if necessary.
Specifically, avoid portals within portals within portals. It's (somewhat) okay
if the different usages of portals are parallel/sibilings,
but nested portals can be especially bad when needing to rerender.
