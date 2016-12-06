- Start Date: 2016-11-22
- RFC PR: (leave this empty)
- Elix Issues: (leave this empty)

# Summary

This proposes a mixin called `SingleSelectionMixin` for components that track a
single selected item at a time. This includes a public API for setting and
retrieving the selected item, as well as methods for moving the selection with
cursor operations.

    // A sample element that supports single-selection on its children.
    class SimpleList extends SingleSelectionMixin(HTMLElement) {
      get items() {
        return this.children;
      }
    }

    const list = new SimpleList();
    list.innerHTML = `
      <div>Zero</div>
      <div>One</div>
      <div>Two</div>
    `;
    list.selectedIndex = 0;
    list.selectedItem // <div>Zero</div>
    list.selectNext();
    list.selectedItem // <div>One</div>
    list.selectedIndex // 1


# Motivation

The abstract concept of single selection is pervasive in user interface
elements. We would like a unified means to implement single selection semantics
in our components for greater consistency, efficiency, and correctness.

Desired outcomes:

1. Components can gain basic single-selection semantics by applying
   `SingleSelectionMixin` and otherwise adding minimal code. A component's
   selection implementation should comply with the relevant Gold Standard
   criteria.
2. Selection semantics remain separate from particular input modalities (e.g.,
   clicking to select something) or output rendering (e.g., always applying a
   particular CSS class like `selected`). This lets developers use the core
   idea of single-selection in a wide variety of situations.
3. The selection of a component is exposed in such a way that it can be easily
   retrieved or set. It should also be possible to bind to the selection in
   frameworks like Polymer that support binding.

Non-goals:

* The initial implementation of `SingleSelectionMixin` is focused on meeting the
  needs of the general-purpose components in Elix. Accordingly, it deals with
  the general notion of "items" that can be selected. There are numerous
  domain-specific components that support selection. Consider an app-specific
  component that lets a user select a friend. The component's API probably
  shouldn't mention "items" in method and property names, but should probably
  mention "friends" instead. Eventually, it should be possible to factor the
  internals of `SingleSelectionMixin` into something that could be used in such
  domain-specific components, but that is not a focus for now.
* The mixin is designed to track selection of HTMLElement objects. As currently
  prototyped, the mixin could actually be used to track selection of other
  objects. Unless it is desirable to preserve that flexibility, however, it will
  be treated as a non-goal. Note also that other Elix mixins will assume that
  the selection is, in fact, an HTMLElement object.


# Use cases

`SingleSelectionMixin` is designed to support components that let the user
select a single thing at a time:

* List boxes
* Dropdown lists and combo boxes
* Carousels
* Tab panels


# Detailed design

## The `items` collection

`SingleSelectionMixin` manages a selection within an identified collection of
HTML elements. A component identifies that collection by defining a read-only
`items` property.

A simplistic implementation of the `items` property might be to return the
component's children:

    class SimpleList extends SingleSelectionMixin(HTMLElement) {
      get items() {
        return this.children;
      }
    }

The above definition for `items` is too simplistic, as it does not support the
[Content Distribution](https://github.com/webcomponents/gold-standard/wiki/Content-Distribution)
criteria, but it can suffice here for demonstration purposes. The key point is
that the component provides the collection of items, and they can come from
anywhere. The component could, for example, indicate that the items being
managed reside in the component's own Shadow DOM subtree:

    class ShadowList extends SingleSelectionMixin(HTMLElement) {
      constructor() {
        // Create shadow root here, populate it with elements.
      }
      get items() {
        return this.shadowRoot.children;
      }
    }

For flexibility, SingleSelectionMixin can work with an `items` collection
defined as either a `NodeList` (as in the above examples) or an `Array`.

### Tracking changes in `items`

If a component wishes `SingleSelectionMixin` to track changes in the `items`
collection, the component must notify the mixin by invoking its `itemsChanged`
method. (That method is referenced via a `Symbol`, and is generally only
available to the component itself.)

A component that is treating its DOM content (including distributed content)
as `items` should invoke `itemsChanged` when that content changes. The
conventional means to detect such DOM changes is for the component to listen
to `slotchange` events on its slots:

    import symbols from './symbols';

    class SimpleList extends SingleSelectionMixin(HTMLElement) {
      constructor() {
        super();
        this.attachShadow({ mode: 'open' });
        const slot = document.createElement('slot');
        this.shadowRoot.appendChild(slot);
        slot.addEventListener('slotchange', event =>
            this[symbols.itemsChanged]());
      }
      get items() {
        return this.children;
      }
    }

This behavior is so common that it will be provided by another Elix mixin.

The `SingleSelectionMixin` performs various checks to maintain a selection when
`itemsChanged` is invoked; see below.


## The selected item

Components managing selection often want to reference the selection in two
different ways: by index and by object reference. `SingleSelectionMixin`
supports both approaches with complementary properties which can _both be
get/set_ and are kept in sync.

* `selectedIndex`. This is the index of the currently selected item within the
  `items` collection. If there is no selection, `selectedIndex` is -1. When this
  property changes, it raises a `selected-index-changed` event.  
* `selectedItem`. This is a reference to the currently selected
  HTMLElement in the `items` collection. If there is no selection,
  `selectedItem` is null. When this property changes, it raises a
  `selected-item-changed` event.

Updating one of these properties also updates the other, as shown above in the
`SimpleList` example at the beginning of this document.

### Updating the selection in response to `itemsChanged`

When the component invokes `itemsChanged`, `SingleSelectionMixin` tracks any
changes the mutations imply for the `selectedIndex` and `selectedItem`
properties. In particular, there are situations in which the selected item might
be moved within the `items` collection. Suppose a list has three items, and
the selected item is moved:

    list.selectedIndex = 0;
    list.appendChild(list.items[0]); // Move selected item to end of list.
    list.selectedIndex // Now this is 2, since item moved.

The above example would raise the `selected-index-changed` event but _not_ the
`selected-item-changed` event, because the `selectedIndex` property changed but
the `selectedItem` property did not.

Similarly, if the selected item is removed from the `items` collection, the
`selectedIndex` and `selectedItem` properties will be reset to -1 and null,
respectively. (Unless `selectionRequired` is true, see below.)

When `SingleSelectionMixin` has finished updating the selection properties,
it raises an `items-changed` event.

### Requiring a selection

The `SingleSelectionMixin` defines a `selectionRequired` property. If true,
the component will always have a selection as long as it has at least one item.
By default, `selectionRequired` is false. This is appropriate, for example, in
components like list boxes or combo boxes which initially may have no selection.

Some components do require a selection. An example is a carousel: as long as the
carousel contains at least one item, the carousel should always show some item
as being selected. Such components can set `selectionRequired` to true.

When `selectionRequired` is true, the following checks are performed when
`itemsChanged` is invoked:

* If items exist and no selection has been established, the first item is
  selected by default.
* If the selected item is removed, a new item is selected. By default, this is
  the item with the same index as the previously-selected item. If that index no
  longer exists because one or more items at the end of the list was removed,
  the last item in the new set is selected.
* If all items have been removed, `selectedIndex` and `selectedItem` are reset
  to -1 and null, respectively.


## Cursor operations

The selection can be programmatically manipulated via cursor methods:

* `selectFirst`. Selects the first item.
* `selectLast`. Selects the last item.
* `selectNext`. Selects the next item in the list. Special case: if no item is
  currently selected, but items exist, this selects the first item.
* `selectPrevious`. Selects the previous item in the list. Special case: if no
  item is currently selected, but items exist, this selects the last item.

If `items` has no items, these cursor operations have no effect.

All cursor methods return a boolean value: true if they changed the selection,
false if not.

### Selection wrapping

In some cases, such as carousels, cursor operations should wrap around from the
last item to the first and vice versa. This optional behavior can be enabled by
setting the mixin's `selectionWraps` property to true. The default value is
false.

### Cursor properties

Two properties track whether the `selectNext` and `selectPrevious` methods are
available:

* `canSelectPrevious`. This is true if the `selectPrevious` method can be
  called. When this property changes, the mixin raises a
  `can-select-previous-changed` event.
* `canSelectNext`. This is true if the `selectNext` method can be called.
  When this property changes, the mixin raises a `can-select-next-changed`
  event.

These properties are useful for components that want to offer, e.g.,
Next/Previous buttons to move the selection. The properties above can be
monitored for changes to know whether such Next/Previous buttons should be
enabled or disabled.

Both `selectNext` and `selectPrevious` support a special case: if there is no
selection but items exist, the methods select the first or last item
respectively. Accordingly, if there is no selection but items exist, the
`canSelectNext` and `canSelectPrevious` properties will always be true.


# Drawbacks

Both the Polymer project (with `iron-selectable-behavior`) and Basic Web
Components (with a mixin along the lines proposed here) have shown that
single-selection can be usefully offered as a core service to components. Hence,
it's hard to imagine _not_ doing anything in this area. In terms of priority,
given the prevalance of single-selection semantics in general-purpose UI
patterns, this seems as good a place as any to begin defining shared Elix
mixins.

The primary concern with `SingleSelectionMixin` might be that leveled against
any other mixin we might begin with: that breaking component behaviors up into
mixins might result in a complex, brittle, or unwieldy entanglements between
mixins. However, given 1.5+ years of experience working with a mixin similar
to `SingleSelectionMixin` indicates that the approach is generally sound.


# Alternatives

What other designs have been considered? What is the impact of not doing this?

requiring dev to expose selection themselves
could factor this later into a mixin with standard API, then internals in
a helper that must be exposed. Allows `selectedUser` instead of `selectedItem`.
