# View Providers

This lesson explains many of the core concepts of Quarve and is hence slightly long.
However, we still recommend reading as it's important!

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
Here, we have a

As another example, while a color (such as `BLACK`) does not know how to control a view,
it knows how to convert itself into a ViewProvider that appropriately displays
a black view. Hence, all colors are IVPs.

### IntoViewProvider
Rather than rewriting every time, we would instead like to instead use.

In general, you will be mainly interacting

Functional

Struct

UpContext and DownContext are related to ways of passing information.
Don't worry about them too much, they are explained more in the next section.

## ViewProvider
Most of the time, you never have to actually implement a full view provider yourself,
but sometimes you may have to implement parts of it (i.e. ). It's also good to know
the general layout model.

The ViewProvider has a few main roles (there are other minor things it does,
such as modifying environment).
1. To create and properly manage the native backing
2. To add subviews and properly lay them out
3. To respond to system and lifecycle events
4. To give proper sizing hints to the parent view so that it can layout this view

### Init Backing

### Layout Model

// also sizes

### Event Handling
