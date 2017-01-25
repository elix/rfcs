- Start Date: 2016-11-22
- RFC PR: https://github.com/elix/rfcs/pull/2
- Elix Issues: https://github.com/elix/elix/pull/2 (implementation PR)

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
    list.selectedIndex = 0;   // Select the first item
    list.selectedItem         // <div>Zero</div>
    list.selectNext();        // Select the next item
    list.selectedItem         // <div>One</div>
    list.selectedIndex        // 1


# Motivation

The abstract concept of single selection is pervasive in user interface
elements. We would like a unified means to implement single selection semantics
in Elix components for greater consistency, efficiency, and correctness.

Desired outcomes:

1. Components can gain basic single-selection semantics by applying
   `SingleSelectionMixin` and otherwise adding minimal code. The resulting
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
  domain-specific components that also support selection. Consider an
  app-specific component that lets a user select a friend. That component's API
  probably shouldn't mention "items" in method and property names, it should
  mention "friends" instead. Eventually, it should be possible to factor the
  internals of `SingleSelectionMixin` into something that could be used in such
  domain-specific components, but that is not a focus for now.
* The mixin is designed to track selection of HTMLElement objects. As currently
  prototyped, the mixin could actually be used to track selection of other
  objects. Unless it is desirable to preserve that flexibility, however, it will
  be treated as a non-goal. Note that most other Elix mixins that deal with
  selection will assume that selectable items are, in fact, HTMLElement
  instances.
* The mixin is not intended at this point for use with infinite sets (such as
  dates in time) or virtualized item collections (e.g., lazy-loaded items in
  large lists). Among other things, the mixin needs to be able to inspect the
  `length` of, and invoke `indexOf` on, the array of items in a performant
  manner.


# Use cases

`SingleSelectionMixin` is designed to support components that let the user
select a single thing at a time. This is generally done to let the user select
a value (e.g., as the target of an action, or in configuring something), or
as a navigation construct (where only one page/mode is visible at a time).
Examples:

* List boxes
* Dropdown lists and combo boxes
* Carousels
* Slideshows
* Tab panels (including top-level navigation toolbars that behave like tabs)


# Detailed design

## The `items` collection

`SingleSelectionMixin` manages a selection within an identified collection of
HTML elements. A component identifies that collection by defining a read-only
`items` property. A simplistic implementation of `items` return the component's
children:

    class SimpleList extends SingleSelectionMixin(HTMLElement) {
      get items() {
        return this.children;
      }
    }

The above definition for `items` is too simplistic, as it does not support the
[Content Distribution](https://github.com/webcomponents/gold-standard/wiki/Content-Distribution)
criteria, but it can suffice here for demonstration purposes. (In practice,
retrieving potentially distributed content as `items` will usually be handled by
another mixin.)

The key point is that the component provides the collection of items, and they
can come from anywhere. The component could, for example, indicate that the
items being managed reside in the component's own Shadow DOM subtree:

    class ShadowList extends SingleSelectionMixin(HTMLElement) {
      constructor() {
        super();
        this.attachShadow({ mode: 'open' });
        // Populate shadow subtree it with elements.
      }
      get items() {
        return this.shadowRoot.children;
      }
    }

For flexibility, SingleSelectionMixin can work with an `items` collection
of type `NodeList` (as in the above examples) or `Array` (e.g., if the
component wants to filter elements for use as items).

### Tracking changes in `items`

If a component wishes `SingleSelectionMixin` to track changes in the `items`
collection, the component must notify the mixin when the set of items has
changed by invoking its `itemsChanged` method. (That method is referenced via a
`Symbol`, and is generally only available to the component itself.)

A component that is treating its DOM content (including distributed content)
as `items` should invoke `itemsChanged` when that content changes. The
conventional means to detect such DOM changes is for the component to listen
to `slotchange` events on its slots:

    import symbols from './symbols';

    class SimpleList extends SingleSelectionMixin(HTMLElement) {
      constructor() {
        super();
        this.attachShadow({ mode: 'open' });
        // Add a single slot, then listen for changes on it.
        const slot = document.createElement('slot');
        this.shadowRoot.appendChild(slot);
        slot.addEventListener('slotchange', event => {
          // Let SingleSelectionMixin know that items have changed.
          this[symbols.itemsChanged]();
        });
      }
      get items() {
        return this.children;
      }
    }

As noted earlier, handling distributed content changes is such a common need
that it will be provided by another Elix mixin.

When `itemsChanged` is invoked, `SingleSelectionMixin` performs several tasks:

1. The mixin determines whether any items are new, and were added since the
   last called to `itemsChanged`. For each new item, the mixin invokes a method
   `itemAdded`, passing in the new item. This allows other mixins to perform
   any per-item initialization. See the ARIA mixin example below in
   "Per-item selection state".
2. If the previously selected item has moved position in the set or has been
   removed, the mixin attempts to preserve the selection; see below.


## The selected item

Components managing selection often want to reference the selection in two
different ways: 1) by index and 2) by object reference. `SingleSelectionMixin`
supports both approaches with complementary properties that can _both_ be
get/set, and that are automatically kept in sync:

* `selectedIndex`. This is the zero-based index of the currently selected item
  within the `items` collection. If there is no selection, `selectedIndex` is -1.
  When this property changes as a result of internal component activity, the
  mixin raises a `selected-index-changed` event.
* `selectedItem`. This is a reference to the currently selected
  HTMLElement in the `items` collection. If there is no selection,
  `selectedItem` is null. When this property changes as a result of internal
  component activity, the mixin raises a `selected-item-changed` event.

Updating one of these properties also updates the other, as shown in the first
`SimpleList` example at the beginning of this document. Setting either property
will raise _both_ the `selected-index-changed` and `selected-item-changed`
events.

# Per-item selection state.

`SingleSelectionMixin` helps map a component's overall selection state (which
item is selected?) to the selection states of individual items (is this item
currently selected or not?). When the `selectedItem` property changes, the mixin
invokes a method, `itemSelected` so that other mixins can update any per-item
selection state. The `itemSelected` method will be invoked either once or twice:

* If there had previously been a selection, the old item is passed to
  `itemSelected` along with a value of `false` to indicate the item is no longer
  selected.
* If there is now a new selected item, the new item is passed to `itemSelected`
  along with a value of `true` to indicate it is now selected.

Example: A simple ARIA mixin could manage the `aria-selected` value for
selected items.

    import symbols from './symbols';

    const SimpleAriaMixin = (base) => class SimpleAria extends base {

      // Mark new items as unselected by default.
      [symbols.itemAdded](item) {
        item.setAttribute('aria-selected', false);
      }

      // Update the aria-selected attribute to reflect a change in item state.
      [symbols.itemSelected](item, selected) {
        item.setAttribute('aria-selected', selected);
      }
    }

This mixin could then be combined with `SingleSelectionMixin` to map selection
semantics to ARIA semantics:

    class SimpleAriaList extends
        SimpleAriaMixin(SingleSelectionMixin(HTMLElement)) {}

    const list = new SimpleAriaList();
    list.innerHTML = `
      <div>Zero</div>
      <div>One</div>
      <div>Two</div>
    `;
    list.children[0].getAttribute('aria-selected') // "false"
    list.selectedIndex = 0;
    list.children[0].getAttribute('aria-selected') // "true"


### Updating the selection in response to `itemsChanged`

When the component invokes `itemsChanged`, `SingleSelectionMixin` tracks any
changes the mutations entail for the `selectedIndex` and `selectedItem`
properties. In particular, there are situations in which the selected item might
be moved within the `items` collection. Suppose a list has three items, and
the selected item is moved:

    list.selectedIndex = 0;
    list.appendChild(list.items[0]);  // Move selected item to end of list.
    list.selectedIndex                // Now this is 2, since item moved.

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
as selected. Such components can set `selectionRequired` to true.

When `selectionRequired` is true, the following checks are performed when
`itemsChanged` is invoked:

* If items exist and no selection has been established, the first item is
  selected by default.
* If the selected item is removed, a new item is selected. By default, this is
  the item with the same index as the previously-selected item. If that index no
  longer exists because one or more items at the end of the list were removed,
  the last item in the new set is selected.
* If all items have been removed, `selectedIndex` and `selectedItem` are reset
  to -1 and null, respectively.


## Cursor operations

The selection can be programmatically manipulated via cursor methods:

* `selectFirst`. Selects the first item.
* `selectLast`. Selects the last item.
* `selectNext`. Selects the next item in the list. Special case: if no item is
  currently selected, but items exist, this selects the first item. This case
  covers list-style components that receive the keyboard focus but do not yet
  have a selection. In such a case, advancing the selection (with, say, the Down
  arrow) can be implicitly interpreted as selecting the first item.
* `selectPrevious`. Selects the previous item in the list. Special case: if no
  item is currently selected, but items exist, this selects the last item. As
  with `selectNext`, this behavior covers list-style components.

If `items` has no items, these cursor operations have no effect.

All cursor methods return a boolean value: true if they changed the selection,
false if not.

### Selection wrapping

In some cases, such as carousels, cursor operations should wrap around from the
last item to the first and vice versa. This optional behavior, useful in
carousel-style components and slideshows, can be enabled by setting the mixin's
`selectionWraps` property to true. The default value is false.

### Cursor properties

Two properties track whether the `selectNext` and `selectPrevious` methods are
available:

* `canSelectPrevious`. This is true if the `selectPrevious` method can be
  called. When this property changes as a result of internal component activity,
  the mixin raises a `can-select-previous-changed` event.
* `canSelectNext`. This is true if the `selectNext` method can be called.
  When this property changes as a result of internal component activity, the
  mixin raises a `can-select-next-changed` event.

These properties are useful for components that want to offer the user, e.g.,
Next/Previous buttons to move the selection. The properties above can be
monitored for changes to know whether such Next/Previous buttons should be
enabled or disabled.

Both `selectNext` and `selectPrevious` support a special case: if there is no
selection but items exist, those methods select the first or last item
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

The primary concern with `SingleSelectionMixin` might be leveled against any
other mixin we might begin with: that breaking component behaviors up into
mixins might result in a complex, brittle, or unwieldy entanglements between
mixins. However, 1.5+ years of experience working with a mixin similar to
`SingleSelectionMixin` suggests that this approach is generally sound.


# Alternatives

We considered packaging standard selection semantics as a helper object, and
requiring the developer to manually expose an appropriate selection API that
delegates selection actions to the helper object. That seems prone to error,
and makes it difficult for component developers to track changes in how Elix
components handle selection. Having a mixin add an appropriate selection API
to the component should allow greater consistency in our general-purpose
components.
