- Start Date: 2016-01-12
- RFC PR: https://github.com/elix/rfcs/pull/4
- Elix code PR: https://github.com/elix/elix/pull/3


# Summary

This proposes a mixin called `SelectionAriaMixin` for list-like components
that want to expose their selection to screen readers and other assistive
technologies via [ARIA](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA)
accessibility attributes. This allows components to satisfy the [Declared Semantics](https://github.com/webcomponents/gold-standard/wiki/Declared-Semantics)
item on the Gold Standard checklist.

Example:

    // A sample element that exposes single-selection via ARIA.
    class AccessibleList extends
        SelectionAriaMixin(SingleSelectionMixin(HTMLElement)) {
      get items() {
        return this.children;
      }
      // Not shown: when items change, invoke [symbols.itemAdded] for new items.
    }
    customElements.define('accessible-list', AccessibleList);

Suppose the developer initially populates the DOM as follows:

    <accessible-list aria-label="Fruits" tabindex="0">
      <div>Apple</div>
      <div>Banana</div>
      <div>Cherry</div>
    </accessible-list>

After the element is added to the page, the DOM result will be:

    <accessible-list aria-label="Fruits" tabindex="0" role="listbox">
      <div role="option" id="_option0" aria-selected="false">Apple</div>
      <div role="option" id="_option1" aria-selected="false">Banana</div>
      <div role="option" id="_option2" aria-selected="false">Cherry</div>
    </accessible-list>

The `SelectionAriaMixin` has selected appropriate default values for the
attributes `role`, `id`, `aria-selected`. When the first item is selected, the
DOM will update to:

    <accessible-list aria-label="Fruits" tabindex="0" role="listbox"
        aria-activedescendant="_option0">
      <div role="option" id="_option0" aria-selected="true">Apple</div>
      <div role="option" id="_option1" aria-selected="false">Banana</div>
      <div role="option" id="_option2" aria-selected="false">Cherry</div>
    </accessible-list>

`SelectionAriaMixin` has updated the `aria-selected` attribute of the selected
item, and reflected this at the list level with `aria-activedescendant`.

In practice, some additional attributes must be set for ARIA to be useful. The
author should specific a meaningful, context-dependent label for the element
with an `aria-label` or `aria-labeledby` attribute. In this example, a
`tabindex` of 0 is also specified, although a planned mixin for general keyboard
support can take care of providing a default `tabindex` value.


# Motivation

Elix mixins and components should support universal access for all users. The
work required to properly expose the selection state of a component in ARIA is
complex, but thankfully fairly generalizable. It should be possible to implement
useful default behavior in a mixin.

Desired outcomes:

1. A component author can easily add ARIA support to a component that uses
`SingleSelectionMixin` or exposes an API consistent with that mixin.

2. All Elix components that expose a selection use the proposed
`SelectionAriaMixin` to support universal access.

3. Because it is trivial to add `SelectionAriaMixin` to a component, a
large number of component developers outside the Elix project use that mixin to
enable ARIA support for their components that support selection.

Non-goals:

* This mixin doesn't address all ARIA needs, just selection.
* The design of this mixin is likely generalizable to multi-selection, but for
  now its is focused on single-selection, as represented in
  `SingleSelectionMixin`. When Elix develops a mixin for multi-selection, we
  should ensure that `SelectionAriaMixin` is updated to work with that.


# Use cases

Use cases for `SelectionAriaMixin` are the same as for
`SingleSelectionMixin`: list boxes, dropdown lists and combo boxes, carousels,
slideshows, tab panels, etc.


# Detailed design

The mixin's primary work is setting ARIA attributes as follows.

`SelectionAriaMixin` complements the model of selection formalized in
the companion `SingleSelectionMixin`. If a component author prefers, they can
skip the latter mixin, and provide their own implementation of the members
`[symbols.itemSelected]`, `[symbols.itemAdded]`, and `selectedItem`.

## `role` attribute on the component and its items

The outer list-like component needs to have a `role` assigned to it. For
reference, the ARIA documentation defines the following [roles] which appear to be
applicable to single-selection elements:

* `combobox`
* `grid`
* `listbox`
* `menu`
* `menubar`
* `radiogroup`
* `tablist`
* `tree`
* `treegrid`

The most general purpose of these roles is `listbox`, so unless otherwise
specified, `SelectionAriaMixin` applies that role by default.

A suitable ARIA role must also be applied at the item level. The default role
applied to items is `option`, defined in the
[documentation](https://www.w3.org/TR/wai-aria/roles#option) as a selectable
item in a list element with role `listbox`.

In situations where different roles are defined, a component can provide a
default value by extending the `[symbols.defaults]` property:

    class TabList extends SelectionAriaMixin(HTMLElement) {
      get [symbols.defaults]() {
        const defaults = super[symbols.defaults] || {};
        defaults.role = `tablist`; // Pick a role for the component
        defaults.itemRole = `tab`; // Pick a role for the items
        return defaults;
      }
      ...
    }

This `defaults` mechanism allows a subclass to override a base class decision
about the default `role` and `itemRole` values.

The mixin applies the role in the `connectedCallback` if the element does not
yet have a `role` attribute. This allows an app to override the `role` on a
per-instance basis by defining a `role` attribute before adding the element to
the page:

    // Letting the mixin pick the role.
    const tabList = new TabList();
    document.appendChild(tabList);
    tabList.getAttribute('role'); // "tablist" (this component's default role)

    // Handling role on a per-element basis.
    const menu = new TabList();
    tabList.setAttribute('role', 'menu');
    document.appendChild(tabList);
    tabList.getAttribute('role')  // "menu" (mixin left the role alone).

The `itemRole` default is applied in the `[symbols.itemAdded]` method, which
the component's implementation must invoke when new items are added. That
responsibility will typically be handled by a mixin to be designed later.


## `id` attribute on the items

ARIA references requires that a potentially selectable item have an `id`
attribute that can be used with `aria-activedescendant` (see below). To that
end, this mixin will generate an `id` attribute for any item added to the list
that doesn't already have an `id`. The mixin performs this work when the
`[symbols.itemAdded]` method is invoked for a new item.

To minimize accidental `id` collisions on a page, the generated default `id`
value for an item includes:

* An underscore prefix.
* The `id` attribute of the outer component, if one has been specified.
* The word "option".
* A unique integer.

Examples: a list with an `id` of `test` will produce default item IDs like
`_testOption7`. A list with no `id` of its own will produce default item IDs
like `_option7`.


## `aria-activedescendant` attribute on the component

To let ARIA know which item is selected, the component must set its own
`aria-activedescendant` attribute to the `id` attribute of the selected item.
`SelectionAriaMixin` automatically handles that whenever the component's
`selectedItem` property is set.


## `aria-selected` attribute on the items

ARIA defines an `aria-selected` attribute that should be set to `true` on the
currently-selected item, and `false` on all other items.

* The mixin sets `aria-selected` to `false` for all new items. This is required
  to adhere to the ARIA spec for roles like
  [tab](https://www.w3.org/TR/wai-aria-1.1/#tab): "inactive tab elements
  [should] have their `aria-selected` attribute set to `false`". That is, it is
  insufficient for an element to omit the `aria-selected` attribute; it must
  exist and be set to `false`. This incurs a performance penalty, as every item
  must be touched by the mixin, but
  [real-world experience](https://github.com/PolymerElements/paper-tabs/issues/176)
  indicates that screen readers do exist which require this behavior.
* When an item's selection state changes, `SelectionAriaMixin` reflects
  its new state in its `aria-selected` attribute. This is done in the
  `[symbols.itemSelected]` method, which is automatically invokes by
  `SingleSelectionMixin`, or the component author can invoke that method
  manually.


# Testing

ARIA features are generally exercised in the context of assistive technologies
integrated with the operating system or browser. The most common such technology
is probably a screen reader. This proposal suggests that Elix generally aim to
have its mixins and components work as expected with the default screen
readers for macOS, Microsoft Windows, iOS, and Android.

That said, testing across screen readers on multiple platforms is challenging.
The project will likely have to rely on a combination of expert review, spot
testing, and user-filed issue reports to achieve its goal.


# Drawbacks

It's hard to think of a reason not to do this.
