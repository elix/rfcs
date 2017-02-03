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

Providing great [Keyboard
Support](https://github.com/webcomponents/gold-standard/wiki/Keyboard-Support)
is a critical Gold Standard checklist item for enabling universal access. It is
also, notably, an area where web developers have historically under-invested.
This can disappoint users of web application, who may be surprised or frustrated
that web UI elements do not measure up to their client OS equivalents.

Desired outcomes:

* Authors have a consistent model for adding keyboard support to their
  components using mixins.
* The Elix list-like components all provide first-class keyboard support.

Non-goals:
* This package of mixins covers general keyboard interaction and navigation of
  lists in particular, but does not cover keyboard support for every situation.
  Other situations will be addressed has they come up in other Elix components.


# Use cases

Use cases for these keyboard-related mixins are the same as for
`SingleSelectionMixin`: list boxes, dropdown lists and combo boxes, carousels,
slideshows, tab panels, etc.


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
a meta key (Command key on Macs, Windows key on Windows) or alt key (Option key
on Macs, Alt key on Windows) are pressed, as that would interfere with browser
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
  the meta key or alt key is pressed, the key is ignored.
* **Right** key: invokes `symbols.goRight` if horizontal navigation is enabled.
  If the meta key or alt key is pressed, the key is ignored.
* **Up** key: invokes `symbols.goUp` if vertical navigation is enabled. If the
  Alt key is pressed, this invokes `symbols.goStart`.

The mixin inspects a property called `symbols.orientation` to determine whether
horizontal navigation, vertical navigation, or both are enabled. Valid values
for that property are "horizontal", "vertical", or "both", respectively. The
default value is "both".


## DirectionSelectionMixin

This mixin maps direction semantics to selection semantics. When a direction
method with a standard identifier is invoked, a corresponding selection method
is invoked:

* `symbols.goDown` → `selectNext`
* `symbols.goEnd` → `selectLast`
* `symbols.goLeft` → `selectPrevious`
* `symbols.goRight` → `selectNext`
* `symbols.goStart` → `selectEnd`
* `symbols.goUp` → `selectPrevious`

A common use of `DirectionSelectionMixin` will be to connect the `KeyboardMixin`
and `KeyboardDirectionMixin` above to the Elix `SingleSelectionMixin`. This
effectively creates a chain of actions that convert keyboard events to changes
in selection. E.g., a press of the Down arrow key would be handled in the
following steps:

1. `KeyboardMixin` receives the `keydown` event for the Down arrow key and
   invokes the component's `symbols.keydown` method.
2. `KeyboardDirectionMixin` handles `symbols.keydown` for the key, and invokes
   `symbols.goDown`.
3. `DirectionSelectionMixin` handles `symbols.goDown` and invokes `selectNext`.
4. `SingleSelectionMixin` handles `selectNext` and updates the selection.

This sequence may seem circuitous, but factoring the behaviors this way allows
other forms of interaction. E.g., a separate mixin to handle touch gestures only
has to map a "swipe left" gesture to a direction method like `goRight` in order
to patch into this chain. This saves the touch logic from having to know
anything about selection.


## SelectionInViewMixin

In scrollable lists, the component may be asked to select an item which is not
currently in view. The `SelectionInViewMixin` listens for changes in the
selection and, if the newly-selected item is not in view, scrolls the component
by the minimum amount necessary to bring the item into view.

For reference, the Blink engine has a `scrollIntoViewIfNeeded()` function that
does something similar, but unfortunately it's non-standard, and often ends up
scrolling more than is absolutely necessary.

The component's scrolling surface may be the component itself, or it may be an
element within the component's shadow subtree. By default,
`SelectionInViewMixin` will try to determine which element should be scrolled.
If the component has a default (unnamed) `<slot>`, the mixin will find the
closest ancestor of that slot that has `overflow: auto` or `overflow: scroll`.
If no such element is found, the component itself will be assumed to be the
scrollable element.

A compnent can also explicitly indicate which of its shadow subtree elements
should be scrolled by defining a property called `symbols.scrollTarget` that
returns the desired element.


## KeyboardPagedSelectionMixin

This mixin supplies standard behavior for the Page Up and Page Down keys,
mapping those to selection operations. The behavior is modeled after that of
standard Microsoft Windows list boxes, which seem to provide more useful
keyboard paging:

* The Page Up and Page Down keys actually change the selection, rather than just
  scrolling. The former behavior seems more generally useful for keyboard users,
  as they can see where the selection is, and can easily refine the selection
  after paging by using the arrow keys.

* Pressing Page Up/Page Down will change the selection to the topmost/bottommost
  visible item if the selection is not already there. Thereafter, the key will
  move the selection up/down by a page.

The `KeyboardPagedSelectionMixin` only updates the selection by setting the
`selectedIndex` property. It does not itself scroll the component. That
responsibility is left to `SelectionInViewMixin` (above).

This mixin relies on `symbols.keydown` being invoked. That will typically be
done with `KeyboardMixin` (above).


## KeyboardPrefixSelectionMixin

This mixin handles list box-style prefix typing, in which the user can type a
string to select the first item that begins with that string.

Example: suppose a component using this mixin has the following items:

    <sample-list-component>
      <div>Apple</div>
      <div>Apricot</div>
      <div>Banana</div>
      <div>Blackberry</div>
      <div>Blueberry</div>
      <div>Cantaloupe</div>
      <div>Cherry</div>
      <div>Lemon</div>
      <div>Lime</div>
    </sample-list-component>

If this component receives the focus, and the user presses the "b" or "B"
key, the "Banana" item will be selected, because it's the first item that
matches the prefix "b". (Matching is case-insensitive.) If the user now
presses the "l" or "L" key quickly, the prefix to match becomes "bl", so
"Blackberry" will be selected.

The prefix typing feature has a one second timeout — the prefix to match
will be reset after a second has passed since the user last typed a key.
If, in the above example, the user waits a second between typing "b" and
"l", the timeout will reset the prefix to empty. When the "l" key is pressed,
"Lemon" will be selected.

If the user presses the Backspace key, the last character is removed from the
prefix under construction. This re-selects the first item that matches the
new, shorter prefix. Continuing the example above: the user types "c", and
"Cantaloupe" is selected, then they type "h" to select "Cherry". If they now
press Backspace (before the aforementioned timeout elapses), the prefix will
revert to "C", and "Cantaloupe" will be reselected.

This mixin expects the component to invoke `symbols.keydown` method when a key
is pressed. This can be provided by `KeyboardMixin` (above).

This mixin also expects the component to provide an `items` property that
returns the list's items. This is the same `items` property used by
`SingleSelectionMixin`.

To extract the text of an item, the mixin invokes a method `symbols.getItemText`
on each item. The default behavior of that method is to return the item's
`alt` attribute or, if that does not exist, its `textContent`. This allows a
component user to customize which text the prefix will be matched against.

For performance, the mixin caches the extracted text of the items. This cache is
invalidated whenever the component or its mixins invoke `symbols.itemsChanged`.


# Drawbacks

The factoring proposed here necessarily introduces some complexity in the name
of reuse. Someone writing only a single component (that is, not interested in
reuse) could conceivably implement that component with less code and very
slightly better performance by optimizing for their situation. E.g., a `keydown`
event handler could listen for the Left arrow key and directly invoke the
`selectPrevious` method.

However, such optimization would be prone to errors. E.g., the developer might
easily and inadvertently trap all Left arrow key presses, unintentionally
disabling the browser's Back button keyboard shortcut while the component had
focus. And most components do have a need to support multiple forms of input. So
optimizing for keyboard support may be premature, and close off important
usability gains that can be realized by using the mixins proposed here.
