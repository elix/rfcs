- Start Date: 2016-11-22
- RFC PR: (leave this empty)
- Elix Issues: (leave this empty)

# Summary

This proposes a SingleSelection mixin for components that track a single
selected item at a time. This includes a public API for setting and retrieving
the selected item, as well as methods for moving the selection with cursor
operations.

    class MyList extends SingleSelection(HTMLElement) {
      get items() {
        return this.children;
      }
    }

    const list = new MyList();
    list.innerHTML = `
      <div>Zero</div>
      <div>One</div>
      <div>Two</div>
    `;
    list.selectedIndex = 0;
    list.selectedItem // <div>Zero</div>
    list.selectNext();
    list.selectedItem // <div>One</div>


# Motivation and goals

The abstract concept of single selection is pervasive in user interface
elements. We would like a unified means to implement single selection semantics
in our components for both consistency and efficiency.

Goals:
* Components can gain basic single-selection semantics by applying the
  SingleSelection mixin and otherwise adding minimal code.
* Avoid entangling selection semantics with particular input modalities (e.g.,
  clicking to select something) or output rendering (e.g., always applying a
  particular CSS class like `selected`).


# Use cases

This mixin is designed to support components that let the user select a single
thing at a time:

* List boxes, including dropdown lists and combo boxes.
* Carousels.
* Tab panels.

# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## The `items` collection

`items`

## The selected item

`selectedIndex`
`selectedItem`
`selected-index-changed`
`selected-item-changed`

## Requiring a selection

`selectionRequired`

## Tracking changes in `items`

`itemsChanged`

`items-changed` event

## Cursor operations

The selection can be manipulated via cursor operations:

* `selectFirst`
* `selectLast`
* `selectNext`
* `selectPrevious`

## Wrapping cursor operations

`selectionWraps`

## Cursor properties

Two properties track whether certain cursor operations are available:

* `canSelectPrevious`
* `canSelectNext`


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