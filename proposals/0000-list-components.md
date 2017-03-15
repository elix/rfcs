- Start Date: 2017-03-14
- RFC PR: (leave this empty)
- Elix Issues: (leave this empty)


# Summary

This proposes an initial collection of Elix components which all have list
characteristics: they support single-selection, and most of their behavior is
defined by the same set of underlying mixins. This proposes three components
of primary interest:

* `ListBox`. A classic single-selection list box.

* `Modes`. Shows exactly one child panel at a time. This does not provide a UI
  of its own, but can be incorporated into, for example, an interactive element
  with multiple modes that present substantially different elements.

* `Tabs`. A set of tabbed panels that can be navigated by selecting
  corresponding tab buttons.

* `LabeledTabs`. A `Tabs` instance with simple tab buttons that have text
  labels.

For flexibility, the `Tabs` and `LabeledTabs` components are composed of some
secondary components:

* `TabStrip`. A row (or column) of tab buttons. Responsible for positioning the
  buttons, keyboard navigation, and supporting accessibility.

* `TabStripWrapper`. Adds a `TabStrip` to a base element, wiring the selection
  states of the two together. For example, the `Tabs` component uses
  `TabStripWrapper` to connect a `TabStrip` to a `Modes` instance.

* `LabeledTabButton`. A classic rounded tab button showing a text label for a
  tab panel. This is used internally by `LabeledTabs` for its tab buttons.

This RFC also proposes some new mixins and helpers:

* `renderArrayAsElements`. Helper function for the common case of rendering
  one element for each item in an array.

* `SelectedItemTextValueMixin`. Exposes a `value` property that gets the
  `textContent` of the `selectedItem`. Can also be used to select an item by
  its `textContent`.

* `ShadowReferencesMixin`. Exposes a `$` member on a component that can be used
  to reference elements in its shadow subtree, effectively behaving like
  Polymer's `$` feature.


# Motivation

Desired outcomes:

1. The primary components in this RFC — `ListBox`, `Modes`, `Tabs`, and
   `LabeledTabs` — find frequent use as helpful UI components.
2. These components serve as references for developers that want to create new
   components which are similar in nature but have unique requirements. Such
   developers can see how this RFC's components are constructed from mixins, and
   then add or subtract from that set of mixins as desired.
3. As the first production-ready web components produced by Elix, these
   components drive real-world requirements for the underlying mixins.

Non-goals:

* Allow extensive customization of `ListBox` and the `Tabs` components via
  properties. These patterns have many variations, and while it is tempting to
  address a large number of them with configurable properties, that generally
  leads to unmanageable complexity.


# Use cases

The primary components in this RFC all have extremely common use cases.

Single-selection lists are very common in user interfaces. As written, the
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
of it as a thing itself. Typically, they will create a containing `div` that
holds an inner `div` for each modal state, then show the `div` for the current
state and hide the rest. While not particularly complex, this comes up so often
in practice that it's efficient to offer as a component.

The `Tabs` component covers the common modal UI pattern in which the user can
directly control which modal state is presented. Tabs are typically used to
allow a UI to offer more controls than can fit in a confined area at a time.
A common use case is Settings or configuration UIs. Tabs may also be used in a
main window to downplay less-commonly used aspects of a UI.

The classic look of a tabbed dialog is addressed with `LabeledTabs`, but the
base `Tabs` class can be used in many other situations. For example, many
mobile applications offer toolbars presenting 3–5 buttons for the app's main
areas. Each toolbar button makes a different set of controls appear. That is,
the navigation model for the mobile application is tabs. Accordingly, such apps
form an important use case for the `Tabs` component.


# Detailed design

The bulk of the functionality provided by `ListBox`, `Modes`, `Tabs`, and
`LabeledTabs` is provided by mixins. The following sections focus on the
API and behaviors which are unique to these components.


## ListBox

This component presents its assigned children as items in a single-selection
list box. This is modeled after the list box controls in macOS and Microsoft
Windows, and the standard `select` element in HTML in its list box mode.

The user can click an item to select it, or navigate the selection
with the keyboard (per `KeyboardDirectionMixin`, `KeyboardPagedSelectionMixin`,
and `KeyboardPrefixSelectionMixin`).

By default, the selected item is shown with standard highlight colors (CSS
`highlight` color for the background, `highlighttext` for the text). This will
eventually be configurable with CSS.

The ListBox exposes an `orientation` property that can either by "vertical"
(the default), or "horizontal".


## Modes

## Tabs

## LabeledTabs

## TabStrip

## TabStripWrapper

## LabeledTabButton


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