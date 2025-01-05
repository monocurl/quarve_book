# Menus

The current interface does not support every possible operation you may want to
do with a menu, but it can do the basics.

## MenuButton
You create the menu tree in the `menu` function of your menu provider. Here,
you typically use the `WindowMenu::standard` function to initialize menus with
standard menu items. You then provide a `Menu`, or list of items, for each category
of File, Edit, View, and Help. In a menu, you can manually specify a new button
as follows.

```rust
fn menu(&self, env: &<Self::Environment as Environment>::Const, s: MSlock) -> WindowMenu {
    WindowMenu::standard(
        env,
        Menu::new("File")
            // here we are explicitly adding our own button
            // with its own custom command
            .push(MenuButton::new("New", "N", EventModifiers::default().set_command(), |_s| {
                println!("Clicked menu button");
            })),
        Menu::new("Edit"),
        Menu::new("View"),
        Menu::new("Help"),
        s
    )
}
```

The problem is that often times you do not explicitly know the action to take upfront,
as it is dependent on the presence of some views in the view hierarchy. Hence, there
is an alternate option as well.

## MenuChannel
The MenuChannel interface for menus also adheres to the sender and receiver pattern.
It may be wise to visit the [portals](./portal.md) lesson first if you haven't already.
The idea is that when you create a window, each menu item is really a receiver
with no action in place. Then, the view hierarchy acts as the sender and provides
the appropriate menu with an action. For instance, we could have a "Select All"
menu item that initially has no action. Whenever a text field is visible, it
enables the "Select All" menu item and populates it with an action when
that menu item is clicked.

The `MenuChannel` is an object that acts as the coordinator between the sender and
receiver. Initially, you create the menu channel object and pass it to the recipient menu
and the view that will act as the sender. Note that `MenuChannel`s
are internally ARCed so that you can clone them without hassle.

Here's how it looks to pass the channel to the menu.
```
fn menu(&self, env: &<Self::Environment as Environment>::Const, s: MSlock) -> WindowMenu {
    // in practice, this initialization would be done
    // somewhere else (likely in environment initialization)
    // so that we can also give the channel to the views
    // but lets ignore that for sake of example
    let new_mc = MenuChannel::new();

    WindowMenu::standard(
        env,
        Menu::new("File")
            .push(MenuReceiver::new(&new_mc, "New", "N", EventModifiers::default().set_command(), s)),
        Menu::new("Edit"),
        Menu::new("View"),
        Menu::new("Help"),
        s
    )
}
```

In general, the easiest way to provide a sender is to use the `menu_send` modifier
of any IVP. Whenever the target IVP is visible, it sends the specified action
to the `MenuReceiver`. When it becomes hidden again, the corresponding action is unset.

```rust
// assume that we properly transferred the same channel as above
// to here
fn rabbits(new_mc: MenuChannel) {
  let new_mc = <...>

  text("rabbits")
    // this will populate the appropriate menu receiver
    // with the given action
    // only whenever the "rabbits" label is shown
    .menu_send(&new_mc, |_s| {
      println!("Creating new rabbit!");
    });
}
```

In some cases however, you may need more fine grained control.
In these cases, you can explicitly use the `set` method of the menu.
Make sure to call `unset` when the action is no longer needed.
