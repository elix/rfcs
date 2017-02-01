- Start Date: 2017-01-31
- RFC PR: (leave this empty)
- Elix Issues: (leave this empty)


# Summary

This proposes a loosely-coupled package of mixins that collectively enable
keyboard support for list-like elements such as list boxes. The user input
aspects of keyboard support (which keys are listened to, and what they do) are
handled separately from their internal or visible effects (what should happen
when those keys are pressed). This allows the mixins to be used in other
combinations or contexts as well.

The proposed package of mixins contains the following:

* **KeyboardMixin**. Wires up a `keydown` event listener that invokes an
  internal `keydown` method. This allows other keyboard mixins (the ones below,
  as well as others that a component author might create) to share a single
  event handler.
* **KeyboardDirectionMixin**. Listens to direction keys (e.g., Up/Down), and
  maps those to semantic direction methods (`goUp`, `goDown`). Those methods can
  be further mapped to selection semantics (see below), or used in other kinds
  of components to easily assign meanings to direction keys.
* **DirectionSelectionMixin**. Maps directional semantics (e.g., `goUp`,
  `goDown`) into selection semantics (`selectNext`, `selectPrevious`). Can be
  used with the above mixin for directional navigation with the keyboard, but
  also with other mixins to support directional navigation with mouse, touch,
  etc.
* **KeyboardPagedSelectionMixin**. Translates page keys (Page Up/Page Down) into
  selection navigation, based on the current visible scroll state of the
  component.
* **SelectionInViewMixin**. Scrolls the component to keep a newly-selected item
  in view. Can be used with the pagination mixin above, or with other kinds of
  selection in scrollable components.
* **KeyboardPrefixSelectionMixin**. Translates prefix typing into selection
  semantics. This allows, e.g., a list box to allow selection by typing the
  start of the desired list item.


# Motivation

What forces are driving us to do this?
What are the desired outcomes?
What are *not* goals?


# Use cases

What use cases does it support?


# Detailed design


## Keyboard


# Drawbacks

Why should we *not* do this? There are tradeoffs to choosing any path, please
attempt to identify them here. Consider the impact on the integration of this
feature with other existing and planned features, on the impact of the API churn
on existing apps, etc.


# Alternatives

What other designs have been considered? What is the impact of not doing this?


# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?