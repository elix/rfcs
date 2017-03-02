- Start Date: 2017-02-28
- RFC PR: (leave this empty)
- Elix Issues: (leave this empty)


# Summary

This proposes a mixin called `AttributeMarshallingMixin` that lets a component
easily map attributes to properties and vice versa. This includes a set of
helpers called `attributes` that components can use to enqueue attribute
changes in their constructor, when direct attribute updates are prohibited.

This sample component defines a property:

    const fooBarSymbol = Symbol('fooBar');

    class MyElement extends AttributeMarshallingMixin(HTMLElement) {
      get fooBar() {
        return this[fooBarSymbol];
      }
      set fooBar(value) {
        this[fooBarSymbol] = value;
      }
    }

    customElements.define('my-element', MyElement);

Because the component applies `AttributeMarshallingMixin`, the camelCase
`fooBar` property can be set in markup via the hyphenated "foo-bar" attribute:

    <my-element foo-bar="Hello"></my-element>

When this element is instantiated, the `fooBar` property setter will
automatically be invoked with the initial value "Hello".


# Motivation

Supporting property initialization via attributes is a fundamental web
component feature. Implementing an `attributeChangedCallback` and an
accompanying static `observedAttributes` getter requires boilerplate code
which is verbose and somewhat cumbersome to keep up to date as new component
properties are added. The `AttributeMarshallingMixin` provides a default
`attributeChangedCallback` that consistently handles this component feature.

Desired outcomes:

* All Elix components consistently handle attributes, so that for every
  property, a corresponding attribute can be set.
* Elix components which need to reflect property values as attributes — e.g.,
  for styling purposes — can do so easily and consistently.

Non-goals:

* This does not parse attribute values. All attributes are passed as strings.
  If a component wants to parse a string attribute value as another data type
  (typically an integer or Boolean value), the propert setter must perform that
  parsing itself.
* This is not intended to support components that dynamically update their own
  public API at runtime.


# Use cases

The use cases for `AttributeMarshallingMixin` are components whose APIs include
public properties — which is to say, essentially all Elix components.


# Detailed design

## AttributeMarshallingMixin

This mixin implements an `attributeChangedCallback` that will convert a change
in an element attribute into a call to the corresponding property setter.
Attributes typically follow hyphenated names ("foo-bar"), whereas properties
typically use camelCase names ("fooBar"). This mixin respects that convention,
automatically mapping the hyphenated attribute name to the corresponding
camelCase property name and invoking the indicated property setter.

Attributes can only have string values, so a string value is what is passed to
the property setter. If you'd like to convert string attributes to other types
(numbers, booleans), you must implement parsing yourself in the property setter.
For example, the following code implements a Boolean property that can be set as
either: a) a Boolean value or b) a string representing a Boolean value:

    get fooBar() {
      return this[fooBarSymbol];
    }
    set fooBar(fooBar) {
      const parsed = String(fooBar) === 'true'; // Cast to Boolean
      const changed = parsed !== this[fooBarSymbol];
      this[fooBarSymbol] = parsed;
      if ('fooBar' in base.prototype) { super.fooBar = fooBar; }
      if (changed && this[symbols.raiseChangeEvents]) {
        const event = new CustomEvent('selection-required-changed');
        this.dispatchEvent(event);
      }
    }

`AttributeMarshallingMixin` also provides a default implementation of
`observedAttributes`. This static getter on the class should return an array of
the attributes the component wishes to monitor. This mixin assumes that the
component wishes to monitor changes in attributes that map to all public
properties in the component's API. E.g., in the above example, the component
defines a property called `fooBar`, so the default value of `observedAttributes`
will automatically include an entry for the hyphenated attribute name,
"foo-bar". A component can override this default implementation
`observedAttributes` if, for some reason, it does _not_ want to monitor changes
in some of its properties. (It is unclear why that would be useful, but that's
up to the developer to decide.)

The mixin also facilitates marshalling property values in the opposite
direction: i.e., changes in properties that should in turn update attributes.
This is not normally necessary, but is useful in specific circumstances: a)
reflecting a property value to an attribute for CSS styling purposes, or b)
updating an ARIA attribute for accessibility purposes. These situations are
addressed with these methods:

* `reflectAttribute`: reflects a value to the component as an attribute.

* `reflectClass`: this specifically handles updates to the `class` attribute,
  and ensures that a given class is (or is not) applied.

This mixin makes use of the `attributes` helpers (below) to allow attributes to
be specified during the component's constructor.


## attributes helpers

These helpers support the setting of attributes (including `class`) during
class construction time.

Consider a component that has a property whose value should be reflected to an
attribute. The Custom Elements specification prohibits the setting of an
attribute in the constructor, so a property that updates an attribute can
likewise not be set during the constructor. Tracking which properties can or
cannot be set in the constructor is burdensome, so it is advantageous to offer a
consistent means of updating attributes that work at any time, including during
execution of the constructor. This is the purpose of the `attributes` helpers.

When these helpers are invoked during the constructor, the helpers queue up
attribute updates. At a later point in time — typically during the
`connectedCallback` — it will be safe to apply attributes, and the helpers can
write out any pending attribute updates. If these helpers are used after the
constructor completes, the attribute updates occur immediately.

The helper functions are:

* `setAttribute`: Sets an attribute to the desired value. If the component has
  not been connected to the document yet, the attribute value is enqueued for
  later application by invoking `connected`.

* `toggleClass`: Like the standard `classList.toggle()` method, this sets,
   clears, or toggles a class. If the component has not been connected to the
   document yet, the class value is enqueued for later application by invoking
   `connected`.

* `writePendingAttributes`: This should be invoked in the component's
  `connectedCallback` to apply any pending attribute or class updates. This can
  also be invoked directly if it is known that the constructor has already
  completed. It is safe to invoke this method more than once; subsequent
  invocations have no effect.

These helpers are provided for the benefit of developers that want to use them
directly. The `AttributeMarshallingMixin` makes use of them in its
`reflectAttribute` and `reflectClass` methods, so developers using that mixin
can use those instead.


# Drawbacks

This mixin enumerates a component's property setters when the browser requests
the value of `observedAttributes` at component registration time. This isn't
necessarily expensive, but is nevertheless work that might not need to be
performed if a given application never makes use of attributes, or only sets
specific attributes.

There may be cases where a component's attribute handling is too complex for
`AttributeMarshallingMixin` and the `attributes` helpers, but in such cases, a
developer is free to dispense with this mixin and handle
`attributeChangedCallback` and `observedAttributes` themselves.
