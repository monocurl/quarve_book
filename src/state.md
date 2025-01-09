# State
This lesson is also a bit long. The important parts are the state lock,
signals, stores and bindings. You can come back to the other parts
after finishing the rest of the lessons.

**Remark:** Some of the terminology of the state system comes from Group theory
(e.g. Word, \[group\] Action).

## State Lock
Most native UI libraries have some operations that can only be performed on the main
thread. Unfortunately, this is often unchecked completely or checked at runtime.
Quarve uses Rust's ownership mechanics to check most of this at compile time.

The idea is that there is a global state lock that only one thread at a time can
hold. Any thread which wants to update or read from the state must acquire the
state lock (often abbreviated 'slock'). To prove that the caller of a certain
function must be holding this lock, you can simply add as an argument
`s: Slock`, which is a marker that can only be acquired if you are in fact the
holder.

There are two types of slocks.
1. `Slock<MainThreadMarker>` which is abbreviated `MSlock`.
2. `Slock<AnyThreadMarker>` which is abbreviated `Slock`.
Note that you can freely convert slocks of the first kind to the second kind
using `.to_general_slock`, which is occasionally necessary. However, converting
the second kind to the first kind fails if you're not on the main thread.

Consequently, many methods in the Quarve library take as argument
either `Slock<impl ThreadMarker>`, denoting only the state lock is necessary,
or `MSlock>`, denoting the state lock is necessary and this function
must be called on the main thread.

**Advanced**: For simple applications, you likely only pass the slock marker
and delegate actually acquiring the lock to the caller of your function
(which will typically be quarve).
However, to explicitly acquire the state locker yourself (as is necessary
for worker threads), you can use the `slock_owner` function
From here, you can get the actual state lock markers using the `marker` function.

**Remark:** Conventional wisdom argues against having a global state lock
as it may reduce parallelism. However, we believe that in the vast majority of cases
only the main thread will hold the lock and worker threads
only need to hold it for a very short time when they perform
the state transformation.
Also, the state lcok greatly reduces race conditions as well.
Finally, we avoid the cost of constantly acquiring and releasing mutexes.
Hence, we believe that in many cases the tradeoff is worth it.

## Signals

Signals are simply values that change over time. Moreover, receivers
can get notified after the value is updated. Note that `Signal`
is a trait, and its value is the associated type Target.

For instance, quarve provides `clock_signal` function that is simply
a signal that continuously increases with time. Here's an example of
how we can apply a listener.
```rust
fn run(s: MSlock) {
    let sig = clock_signal(s);
    sig.listen(|val, s| {
        println("Time is {}", val);
        true
    }, s);

    // you can also query the current value of a signal
    // using the borrow method
    // note that this requires the state lock as well
    let curr = *sig.borrow(s);
}
```

Notice that signal listeners are given the state lock as a parameter,
and you can only add listeners if you have the state lock.
Listeners also should return a boolean value denoting whether they
want to continue listening or should be dropped.
In the above example, we want to listen forever so we always
return `true`. In many cases, you return false once a weak
reference can no longer be upgraded.

By default, `listen` gives a notification for all possible updates,
even when the 'new' value is the same as the old one.
If you would only like to get notifications, you can use
`diff_listen` instead (this requires the target to implement `PartialEq`).

## Store
The heart of state is a `Store`. Conceptually, a store simply
stores a particular value and notifies to its observers whenever it changes.
In this aspect, it's a signal. However, unlike a signal, a `Store` can also
be changed.

In the simplest case, this change is done by manually overwriting the current value.
```rust
let store = Store::new(0);
// notice the usage of the state lock
// Here, we apply the SetAction::Set to modify the current value
store.apply(Set(4), s);

// you can acquire a signal from a store explicitly
// sig: impl Signal<Target=i32> + Clone
let sig = store.signal();

// or... you can add listeners directly to the store
store.listen(|val, s| {
    ...
    true
}, s);
```

The advantage of having the action be an actual struct passed in
(as opposed to e.g. some function on the object)
is that now observers can see the exact transformation that was
applied, which can be surprisingly useful.

For many types, such as integers and floats, the 'action' of how you
change the value is simply setting it. For instance, the action for a vector
is a sequence of inserts or removals.
```rust
let store = Store::new(vec![Store::new(false)]);

// for store of vectors,
// it may not be obvious why we need the inner store
// (in addition to the outer one)
// you can think about it as the top level store lets us observe
// if elements are added or deleted, but doesn't know if they're modified
// the individual sub stores allow observes to see if any particular
// element of the vector is modified
store.apply([
    VecActionBasis::Insert(Store::new(false), 0),
    VecActionBasis::Remove(1)
], s);
```

Again, the advantage is that now an observer is not only notified that
the vector will change, but they can see exactly *how* it will be changed
by looking at the action. This can, for example, figure out how to efficiently
undo the action or possibly mirror the action onto another vector.

## Binding
In particular, in addition to being a signal, stores also implement
the `Binding` trait. The `Binding` trait is subtrait of `Signal`,
further specifying that you must be able to change the current
value using an action (as was seen above), and add an `action_listener`.

An action listener gets notified every time a store *is about to* change
(in contrast, regular listeners are notified afterwards). It is given
the current value, the action, and the state lock.
```rust
let store = Store::new(vec![false, true]);
// see below for why the different syntax wrt to signal
// b: impl Binding<Filterless<Vec<bool>> + Clone
let b = store.binding();

// here we call action listen on the binding
// but you could've just called it on the store itself
b.action_listen(|val, action, s| {
    /* act appropriately */

    // again, return a value based on whether we want to continue
    // listening
    true
}, s);

```

Finally, we note that `Store` implements `Binding` but does not
implement `Clone`. If you would like a cloneable binding,
explicitly call the `binding` method on the store.
(likewise, for a cloneable signal, use the `signal` method).

Also note that Stores are internally arc-ed. In particular,
even if the original `Store` is dropped but one of the bindings
is still alive, you can still use the bindings as they hold strong references.
You can instead call `weak_binding` to only hold a weak reference.

**Note:** There is a slight asymmetry between `impl Signal<Target=T>`
and `impl Binding<Filterless<T>>`.
The filterless part is basically for something
that didn't really end up getting used but is still a part of quarve.
It may be used in the future, though.

## Store Container
In any actual UI application, the main state is not simply a single
store, but rather a collection of them. We would like to organize such stores
into containers. The `quarve_derive` crate provides a utility to automatically
derive StoreContainer.

```rust
#[derive(StoreContainer)]
struct ApplicationState
    // non stores should be ignored
    #[quarve(ignore)]
    app_name: String,

    count: Store<usize>,
    shopping_cart: Store<Vec<Item>>,
    // store containers can hold other store containers
    // (assume that UserState is some other StoreContainer)
    user: UserState
}
```

In addition to grouping together state, StoreContainers provide
the the functionality of adding a listener to see when
*any* contained store changes. As we'll soon see, StoreContainers also are
used for adding undo/redo support.

```rust

let a = ApplicationState::new(/* snip */);
// note that only one subtree general listener
// can ever be present at any given time
a.subtree_general_listener(|s| {
    // note that we are not explicitly told which
    // sub container is changed
    true
}, s);
```

Quarve also provides `StoreContainerSource` and `StoreContainerView`
to add arc funtionality to store containers, if needed.

## Undo
Quarve makes it easy to add undo support. Place all stores that
should be undoable under some `StoreContainer`. Then, simply
create an `UndoManager` and call `mount_undo_manager` on
the target IVP. Whenever the IVP is mounted on the view
hierarchy (i.e. visible), you will be able to undo any action.

```rust
fn root(&self, env: &<Env as Environment>::Const, s: MSlock) -> impl ViewProvider<Env, DownContext=()> {
    let a = ApplicationState::new(/* snip */);
    // specify the store container
    let u = UndoManager::new(&a, s);

    ivp
        .mount_undo_manager(u)
        .into_view_provider(env, s)
}
```

**TODO (advanced) mention undo grouping, undo bucket and history_elide**

## Derived Store
**TODO add more details here**
However, sometimes we want to explicitly exclude a Store
from undo operations. This usually happens because a Store is not
actually independent of other stores, but is instead a function of them
(i.e. its value is 'derived' from other stores).
Concretely, let's suppose that we have two stores. One is a counter,
and another is a double counter that always
(toy example, you can imagine that actual conversion
to be much more complex).
```rust
let a = Store::new(0);
let b = Store::new(0);

// always update b
let binding = b.binding();
a.listen(move |val, s| {
    binding.apply(Set(2 * val), s);
    true
}, s);
```

In such a case, let's suppose somewhere that `a` is updated
to `1`. Following this, `b` will be updated to `2`.
When we request an undo, we will revert the most recent action,
so `b` will go back to `0`. Then, we will set back `a` to `0`.
Since `a` was changed, all listeners will be invoked,
so `b` will again be set to `0`. This last operation is redundant,
and we could've simply gotten away with never undoing
the action for b in the first place. Hence, it makes more sense
to mark `b` as a DerivedStore that should not partake
in undo operations.

You can imagine this becomes much more problematic when `b` is
trying to mirror the actions of `a` (that aren't simply SetActions,
think of a vector).
In this case, applying the inverse action twice will lead
to incorrect behavior!


## Stateful
How does quarve know which action type shoul be used for a given store?
The answer is that any item used in the store must implement
the `Stateful` trait. Part of the `Stateful` trait is
setting the associated type to the natural action.

Hence, if you want to create a store of some custom type,
you must implement `Stateful` for the your custom type.
For many cases, you can get away by setting the action
to be `SetAction`. If necessary, you can instead create
your own action and  implement the `GroupAction` trait
for it.

\* Technically, it need not be a 100% proper
group action, but it's nearly there.

## IntoAction
Sometimes, we may want to perform a modification that
isn't naturally thought of as the standard action for
some Store. For example, let's suppose we have a counter.
It's quite annoying to use a set action to increment
since we must first borrow the value.

Hence, we note that `Binding::apply` actually accepts
a value that implements `IntoAction`, rather than solely
the standard action of the `Store`. Consequently, we can create
an `IncrementAction` struct that implements `IntoAction` and
be converted into a `SetAction`. Crucially, note that in the below
signature, we are allowed to look at the target value when performing
the conversion so that we may simply return `SetAction::Set(1 + *target)`.

```rust
fn into_action(self, target: &T) -> A;
```

In fact, `NumericAction::Incr` exists in the quarve library
and does exactly this.
You can feel free to create your own too though, as you see fit.

Another use case is to simply give syntatic sugar for actions
that are cumbersome to type. For instance,
`[VecActionBasis; N]` implements `IntoAction` so that
you can type `[VecActionBasis::insert(...)]` instead of
`Word::new(vec![VecActionBasis::insert(...)])`.
