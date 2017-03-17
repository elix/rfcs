- Start Date: 2017-03-14
- RFC PR: (leave this empty)
- Elix Issues: (leave this empty)


# Summary

This proposes the creation of two components:

* `ListBox` for simple single-selection lists.

* `Modes`. Shows exactly one child panel at a time. This does not provide a UI
  of its own, but can be incorporated into, for example, an interactive element
  with multiple modes that present substantially different elements. `Modes`
  turns out to share much of the same functionality as `ListBox`, hence its
  inclusion in this proposal.

This RFC also proposes a new mixin to support `ListBox`:

* `SelectedItemTextValueMixin`. Exposes a `value` property that gets the
  `textContent` of the `selectedItem` and can also be used to select an item by
  `textContent`.


# Motivation

Desired outcomes:

1. The `ListBox` and `Modes` find frequent use in their own right.
2. These components serve as references for developers that want to create new
   components which are similar in nature but have unique requirements. Such
   developers can see how this RFC's components are constructed from mixins, and
   can reuse those pieces to suit their needs.
3. As the first production-ready web components produced by Elix, these
   components drive real-world requirements for the underlying mixins.

Non-goals:

* Allow extensive customization of these components via properties. These
  patterns have many variations, and while it is tempting to address a large
  number of them with configurable properties, that generally leads to
  unmanageable complexity.


# Use cases

The two components in this RFC all have extremely common use cases.

Single-selection list boxes are common in user interfaces. As written, the
`ListBox` component supports the same use cases as a standard `select` element
in list box mode (with the `size` attribute set to a value greater than 1). The
standard `select`, however, has many constraints on what can be done with it,
forcing developers to recreate much of its functionality. The advantage of
providing `ListBox` as a web component is that developers are free to extend it
to meet the needs of their interface. For example, one use case for `ListBox`
is a horizontal list (instead of the fixed vertical orientation provided by
`select`).

The use cases for `Modes` are similarly ubiquitous: any portion of a UI with
a modal state that can display two or more different sets of UI controls
depending upon the state. This pattern is so common that developers rarely think
of it as a thing itself. Historically, they have often created a containing
`div` holding an inner `div` for each modal state, then show the `div` for the
current state and hide the rest. But rewriting that code each time is slow and
error-prone. `Modes` serves this need in much the same way as Polymer's
`iron-pages` component.


# Detailed design

## ListBox component (elix-list-box)

This component presents its assigned children as items in a single-selection
list box. This is modeled after the list box controls in macOS and Microsoft
Windows, and the standard `select` element in HTML in its list box mode.

`ListBox` uses `SingleSelectionMixin` to expose a single selection. The user can
click an item to select it, or navigate the selection with the keyboard (per
`KeyboardDirectionMixin`, `KeyboardPagedSelectionMixin`, and
`KeyboardPrefixSelectionMixin`).

By default, the selected item is shown with standard highlight colors (CSS
`highlight` color for the background, `highlighttext` for the text). This will
eventually be configurable with CSS, although Elix is still working out a
general theming strategy.

The ListBox exposes an `orientation` property that can either by "vertical" (the
default), or "horizontal". Moreover, the list reflects its `orientation`
property value as an attribute to simplify conditional styling.

The ListBox applies `SelectedItemTextValueMixin` (below) to expose a `value`
property.

By default, a `ListBox` shows a generic visual style. Once the Elix project
establishes a theming strategy, we will allow developers to style a `ListBox`
instance with CSS.


## Modes component (elix-modes)

The `Modes` component shows exactly one of its assigned children at any given
time. It can be thought of as a list that only shows the currently selected
item.

`Modes` uses `SingleSelectionMixin` to expose a single selection. A developer
must programmatically modify which item is currently visible, as the component
provides no user interface itself.


## SelectedItemTextValueMixin

This mixin adds a `value` property that gets the `textContent` of a component's
`selectedItem`.

The `value` property can also be set to set the selection to the first item in
the `items` collection that has the requested `textContent`. If the indicated
text is not found in `items`, the selection is cleared.

Naming issue: The name `value` has been deliberately chosen to mirror the
`value` property of a standard `input` element. However, given the precedence of
the `selectedItem` and `selectedIndex` property names, we could also call this
property `selectedText`.


# Unresolved questions

The `Modes` component only shows a single panel at a time; the other panels are
hidden by setting `display: none`. What is unclear is whether those hidden
panels should still be hidden to ARIA by applying the `aria-hidden` attribute.
Sometimes this will be desirable, as when an inactive mode should be both
physically invisible and invisible to ARIA. In other cases, it might be
desirable to let the user navigate the modes with the keyboard, in which case
ARIA should be able to see the inactive modes. We may need to expose that option
as a property, or factor `Modes` into two components, or rely on authors
applying `aria-hidden` themselves.

Also see the naming issue raised for `SelectedItemTextValueMixin`.
