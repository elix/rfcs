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

## KeyboardMixin

This mixin lets a component and multiple mixins all handle keyboard events with
a single `keydown` event listener, with the general intention of supporting
keyboard navigation of a component's main features.

For this reason, a `keydown` listener is used rather than `keyup`, as most
keyboard behavior (e.g., pressing arrow keys) should respond on keydown for
faster response time and to allow key repeats. A `keydown` event is used instead
of `keypress` because `keypress` is not fired when many keys are pressed,
including Tab, Delete, backspace, arrow keys. (See this [webkit
changelog](https://lists.webkit.org/pipermail/webkit-dev/2007-December/002992.html)
for more details.)

When `KeyboardMixin` receives a keydown event, it invokes a method called
`symbols.keydown`, passing in the event object. The component or its other
mixins can then take their turns examining the event. An implementation of
`symbols.keydown` should return `true` if it handled the event, and `false`
otherwise.

The convention for handling `symbols.keydown` is that the last mixin applied
wins. That is, if an implementation of `symbols.keydown` *did* handle the event,
it can return immediately. If it did not, it should invoke `super` to let
implementations further up the prototype chain have their chance.

Example:

    class EnterElement extends KeyboardMixin(HTMLElement) {
      [symbols.keydown](event) {
        if (event.keyCode === 13 /* Enter */) {
          console.log("Enter key was pressed.");
          return true; // We handled the event.
        }
        // We didn't handle the event; invoke super.
        return super[symbols.keydown] && super[symbols.keydown](event);
      }
    }

If an implementation of `symbols.keydown` returns `true` — indicating that the
event was handled — then `KeyboardMixin` invokes the event's `preventDefault`
and `stopPropagation` methods to let the browser know the event was handled.


## KeyboardDirectionMixin

This mixin maps arrow and Home/End keys to semantic direction methods. This
allows a developer to quickly support directional navigation in a component.
This also helps avoid common pitfalls. For example, a component listening to
presses of the Left or Right keys should take care to _not_ handle those keys if
the meta (Command) or Alt keys are pressed, as that would interfere with browser
navigation shortcuts.

This mixin relies on `symbols.keydown` being invoked. The typical way that will
be done is with `KeyboardMixin` (above), but a component author can of course
handle keyboard events and invoke the method themselves.

The supported mapping of direction keys to direction methods is:

* **Down** key: invokes `symbols.goDown` if vertical navigation is enabled. If
  the Alt key is pressed, this invokes `symbols.goEnd`.
* **End** key: invokes `symbols.goEnd`.
* **Home** key: invokes `symbols.goStart`.
* **Left** key: invokes `symbols.goLeft` if horizontal navigation is enabled. If
  the meta (command) or Alt key is pressed, the key is ignored.
* **Right** key: invokes `symbols.goRight` if horizontal navigation is enabled.
  If the meta (command) or Alt key is pressed, the key is ignored.
* **Up** key: invokes `symbols.goUp` if vertical navigation is enabled. If the
  Alt key is pressed, this invokes `symbols.goStart`.


## DirectionSelectionMixin


## KeyboardPagedSelectionMixin


## SelectionInViewMixin


## KeyboardPrefixSelectionMixin


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