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
you will more likely implement the **IntoViewProvider** trait.

Let's take a concrete example to highlight the distinction.
Here, we have a

As another example, while a color (such as BLACK) does not know how to control

## Beginner

ViewProviders.

### IntoViewProvider
Rather than rewriting every time, we would instead like to instead use.

In general, you will be mainly interacting

Functional

Struct

UpContext and DownContext are related to ways of passing information.
Don't worry about them too much, they are explained more in the advanced section.

## Advanced

We recommend you come back to this section once you have explored more of Quarve.
