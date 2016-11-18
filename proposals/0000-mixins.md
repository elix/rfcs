- Start Date: 2016-11-17
- RFC PR: (leave this empty)
- Elix Issues: (leave this empty)


# Summary

This proposes that the project agree upon functional mixins as our standard way
to share aspects of behavior and public APIs across our components. Functional
mixins take the form of a function that takes a class argument and returns an
extended class implementing the mixin's functionality:

    const MyMixin = (base) => class MyMixin extends base {
      // Mixin defines properties and methods here.
      greet() {
        return 'Hello';
      }
    };

    class MyElement extends MyMixin(HTMLElement) {}

    let obj = new MyElement();
    obj.greet(); // Hello


# Motivation and use cases

Web components often share aspects of behavior and public API. For example, many
components support the notion of single selection, where the user can pick at
most one item from the component's children. It is advantageous to implement
such behavior in a reusable fashion at the level smaller than that of a complete
component.

Many UI component frameworks support some form of mixin, although the exact
semantics of mixins can vary from framework to framework. That increases
learning costs and hinders reuse. Functional mixins provide a greater degree of
interoperability, because they directly leverage the JavaScript prototype chain,
and so can be used with a variety of base classes.

Design goals:

1. **Allow mixins that can focus on a single, common component task.**
   Each mixin should be useful on its own, or in combination.
2. **Introduce as few new concepts as possible.**
   A developer who understands the DOM API should be able to work with these
   mixins without having to substantially change the way they write code. They
   shouldn't have to learn anything new beyond the concept of defining a mixin
   as a function.
3. **Anticipate native browser support for ES6 and web components.**
   The architecture should be usable in an ES5 application today, but should
   also feel correct in a future world in which native ES6 and web components
   are everywhere.

Use cases for functional mixins for web components:

* Template stamping. A component would like to create a new shadow root in the
  component `constructor` and clone a template into it.
* Attribute marshalling. A component would like to implement a default
  `attributeChangedCallback` to marshall hyphenated `foo-bar` attributes to
  the corresponding camelCase `fooBar` properties.
* Single selection. A component would like to implement standard single-selection
  semantics, with a `selectedItem` property, `selectNext`/`selectPrevious` methods,
  and so on.
* ARIA list semantics. A component that supports selection (above) would like to go
  further and expose the currently selected item via ARIA attributes to make the
  component accessible to users of, e.g., screen readers.

These mixins can be applied directly to the standard `HTMLElement` class if a
developer wants to work directly on top of the platform. These mixins are also
designed to be used with component base classes from web component frameworks.
Significantly, this functional mixin approach is consistent with that adopted by
Google's Polymer team for their future releases. Their earlier, proprietary
"behaviors" will be replaced with mixins such as those described here.

This mixin strategy is designed to mitigate common problems with earlier mixin
approaches, for example the mixin approach which is now  
[deprecated in React](https://facebook.github.io/react/blog/2016/07/13/mixins-considered-harmful.html).
The mixin strategy described here avoids a number of the problems with classical
mixins.

1. The functional mixins described here are closer in spirit to what React calls
   "higher-order components" than traditional JavaScript mixins.
2. Mixins are only used to provide behavior that affects or interacts with the
   component's public API.
3. Naming conflicts are resolved using the JavaScript prototype chain, and a
   mixin can invoke base class functionality via `super`.
4. Naming conflicts are further avoided by using property symbols, not string
   names, to access properties and methods that do not need to be exposed in the
   public API.

While this mixin strategy is sufficiently general to be of interest to problem
domains outside of web component creation, this plan is focused only on creating
web components in the context of the Elix project. Theoretical weaknesses that
are not likely to apply to Elix components may therefore not be significant
practical concerns.

The expected outcome of this design:

* The creation of a well-factored collection of mixins for all behavior aspects
  shared across Elix project components. Most of the components will be
  primarily comprised of mixins, with very little code required to define the
  final, instantiable custom element.
* A developer that wants to use an Elix component, but does *not* want some
  aspect of its behavior, can create a new component that uses the same set of
  mixins as the original, minus the mixin(s) they want to leave out.
* The same mixins can be used by developers on other projects to help them
  create components which meet the Gold Standard. Ideally, they can use the
  mixins in any framework that permits the creation of web components in
  JavaScript using `class` syntax. The mixins permit a pay-as-you-go approach
  to web component complexity and performance.


# Detailed design

## Mixins as functions

The mixins in this project all take the form of a function. Each function takes
a base class and returns a subclass defining the desired features:

    const MyMixin = (base) => class MyMixin extends base {
      // Mixin defines properties and methods here.
      greet() {
        return "Hello";
      }
    };

    class MyBaseClass {}
    const NewClass = MyMixin(MyBaseClass);

    const obj = new NewClass();
    obj.greet(); // "Hello"

Many JavaScript mixin implementations destructively modify a class prototype,
but mixins of the functional style shown above do not. Rather, functional mixins
extend the prototype chain and return a new class. Such functions have been
called "higher-order components", but we prefer the term "mixin" for brevity.

The mixins in this project take care to ensure that base class properties and
methods are not broken by the mixin. In particular, if a mixin wants to add a
new property or method, it also invokes the base class' property or method. To
do that consistently, these mixins follow standardized composition rules
(below). If you are interested in creating your own component mixins, you may
find it helpful to follow those guidelines to ensure that your mixins can
interoperate cleanly with the ones in this project.

A core virtue of a functional mixin is that you do not need to use any library
to apply it. This lets you use these mixins with any conventional means of
defining JavaScript classes — you don't have to invoke a proprietary class
factory, nor do you have to load a separate framework or runtime.

Because mixins define behavior through composition, you're not limited by the
constraints of a single-inheritance class hierarchy. That said, you can still
use a class hierarchy if you feel that's suitable for your application. For
example, you can compose a set of mixins to create a custom base class from
which your other classes derive. But the use of such a base class is not
dictated here.


## Mixin convention

Mixins in this project are named with an initial capital: `MyMixin`, and not
`myMixin`. The initial capital is intended to suggest their status as
partial quasi-classes.

The mixin name will generally appears twice in mixin definitions such as that
shown earlier:

    const MyMixin = (base) => class MyMixin extends base { ... }

The first use of `MyMixin` is the name of the function that can be applied to a
class to get back an extended class. The second `MyMixin` is actually not
required, but is helpful for debugging. It allows the debugger to show you a
meaningful name when you inspect a component's prototype chain at runtime.


## Semantic mixin factoring

A core intention behind the use of mixins for this project is to factor
component services into separable but complementary mixins. This loose
arrangement will permit easier code maintenance (a closer correspondance...

  ***

In a number of areas, this package factors high-level component services into
mixins that work together to deliver the overall service. This is done to
increase flexibility.

For example, this library includes three mixins that work in concert. When
applied to a custom element, these mixins take care of mapping presses on
keyboard arrows (Left/Right) into selection actions (select previous/next).
They each take care of a different piece of the problem:

* The [Keyboard](docs/Keyboard.md) mixin wires up a single keydown listener on
  the component that can be shared by multiple mixins. When the component has
  the focus, a keypress will result in the invocation of a `keydown` method. By
  default, that method does nothing.
* The [KeyboardDirection](docs/KeyboardDirection.md) mixin maps keyboard
  semantics to direction semantics. It defines a `keydown` method that maps
  Left/Right arrow key presses into calls to methods `goLeft` and `goRight`,
  respectively. By default, those methods do nothing.
* The [DirectionSelection](docs/DirectionSelection.md) mixin maps direction
  semantics to selection semantics. It defines `goLeft` and `goRight` methods
  which respectively invoke methods `selectPrevious` and `selectNext`. Again, by
  default, those methods do nothing.

If all three mixins are applied to a component, then when the user presses, say,
the Right arrow key, the following sequence happens:

    (keyboard event) → keydown() → goRight() → selectNext()

Other mixins can map selection semantics to user-visible effects, such as
highlighting the selected item, ensure the selected item is in view, or do
something entirely new which you define.

Such factoring may initially feel overly complex, but permits a critical degree
of developer freedom. You might want to handle the keyboard a different way,
for example. Or you may want to create a component that handles arrow
keypresses for something other than selection, for example. Or you may want to
let the user manipulate the selection through other modalities, such as touch
gestures, mouse actions, speech commands, etc.

As one example of another mode of user input, the
[SwipeDirection](docs/SwipeDirection.md) mixin maps touch gestures to `goLeft`
and `goRight` method calls. It can therefore be used in combination with the
DirectionSelection mixin above, with the result that swipes will change the
selection:

    (touch event) → goRight() → selectNext()

The SwipeDirection and KeyboardDirection mixins are compatible, and can be
applied to the same component. Users of that component will be able to change
the selection with both touch gestures and the keyboard.

This factoring allows components with radically different presentations to
nevertheless share a considerable amount of user interface logic. For example,
the [basic-carousel](../packages/basic-carousel) and
[basic-list-box](packages/basic-list-box) components look very different, but
both make use of same mixins to support changing the selection with the
keyboard. In fact, nearly all of those components' behavior is defined through
shared mixins factored this way.


## Conventions for extending base class methods and properties in a mixin

Mixins functions that extend a class by creating a subclass should generally
*extend* base class methods and properties, not replace them.

The default behavior in JavaScript is for subclass members to override members
with corresponding names in base classes. That behavior may be useful in cases
where a subclass is extending a known, fixed base class. In those cases, the
subclass author generally knows what the base class can do, and can invoke base
class behavior if desired via `super`. Depending on knowledge of the base class,
the subclass author may elect not to invoke the base class implementation.

However, mixins may be applied to a variety of base classes. Mixin authors
should generally write methods and properties that augment base class
functionality rather than replacing it. This increases the chance that base
class functionality will continue to work even in the presence of mixins.

Generally, this means that mixin methods and properties should invoke `super`.
Unlike the normal application of `super` in subclasses, mixins do not have any
guarantee that a base class actually implements a given method or property.
Hence, the mixin cannot just blindly invoke `super` — it must generally inspect
the base class first for the presence of the method or property and, only if it
exists, invoke `super`.

There is no single solution that works for all kinds of mixin members. Instead,
you can follow the series of rules presented below.


### Method known to not return a result

This is the simplest case. Your mixin method should check to see whether the
base defines a method of the same name, and if so, invoke that method. In most
cases, you should invoke the super method before performing your own mixin work.

    const Mixin = (base) => class Mixin extends base {
      method(...args) {
        if (super.method) { super.method(...args); }
        // Do your mixin work here.
      }
    };

Be sure to pass along any arguments to the base class’ method implementation.


### Method that may return a result

If the base class method might return a result, you should capture that result
before doing mixin work. Your mixin has the opportunity to modify the base
method’s result, or can leave it unchanged. In any event, your mixin method
should itself return a result.

    const Mixin = (base) => class Mixin extends base {
      method(...args) {
        let result = super.method && super.method(...args);
        // Do your mixin work here, potentially modifying result.
        return result;
      }
    };


### Property getter with no setter

If you’re certain that a property will *never* have a setter, you can omit
defining a setter. But if any other mixin might define a setter for the
property, you need to define one, too. Otherwise the absence of a setter will
implicitly override a setter further up the prototype chain.

To avoid this problem, define a default setter that checks to see whether it
should invoke super. To check the base class for the existence of a property,
use the idiom, `'name' in base.prototype`.

Your getter will generally want to override the base class implementation, so it
does not need to invoke super.

    const Mixin = (base) => class Mixin extends base {
      get property() {
        // Do your mixin work here.
      }
      set property(value) {
        if ('property' in base.prototype) { super.property = value; }
      }
    };

Note the use of `base.prototype` instead of `super` in the check — `'name' in
super` is not legal ES6. However, the use of `super` *is* required in the
actual `super.prototype = value` assignment which follows. Using
`base.prototype` there wouldn’t work: it would overwrite the base’s setter
instead of invoking it.


### Property setter with no getter

It’s possible that your mixin will want to define a setter but not a getter:
your mixin may want to do work when the property changes, but leave the actual
retrieval of the property to a base class.

In such situations, you must still supply a default getter that invokes super.
(Otherwise, the absence of a getter in your mixin will implicitly override a
getter defined further up the prototype chain.) Your getter does not need to
check to see whether it should invoke super — if the base class doesn’t define
the property, the result will be undefined anyway.

    const Mixin = (base) => class Mixin extends base {
      get property() {
        return super.property;
      }
      set property(value) {
        if ('property' in base.prototype) { super.property = value; }
        // Do your mixin work here.
      }
    };


### Property with both getter and setter

This is a combination of the above two rules. The getter will generally want to
override the base class property, so it doesn’t need to invoke super. The setter
should use the same technique above to see whether it should invoke super.

    const Mixin = (base) => class Mixin extends base {
      get property() {
        // Do your mixin work here.
      }
      set property(value) {
        if ('property' in base.prototype) { super.property = value; }
        // Do your mixin work here.
      }
    };


### Property with a getter and setter that "backs" a value

A common case of the above rule is a mixin that will store the value of
property for the benefit of other mixins and classes. Such a mixin is said to
"back" the property.

In this situation, the recommendation is to have the setter record the new value
of the property *before* invoking `super`. This ensures that, if the superclass
immediately inspects the property's value, the latest value will be returned.
Other work of the mixin should be done *after* invoking `super`, as usual.

    let propertySymbol = Symbol();

    const Mixin = (base) => class Mixin extends base {
      get property() {
        return this[propertySymbol];
      }
      set property(value) {
        this[propertySymbol] = value;
        if ('property' in base.prototype) { super.property = value; }
        // Do any other mixin work here.
      }
    };


## Mixin order independence

A critical goal is designing web component mixins which can be applied in any
order. That is, if `Foo()` and `Bar()` are functional mixins, then both
`Foo(Bar(HTMLElement))` and `Bar(Foo(HTMLElement))` should effectively behave
the same. Order independence makes it much easier for developers to use mixins.

The general nature of this mixin architecture does not guarantee order
independence: two people could easily write two mixins that modify the same
element attribute, in which case one mixin might overwrite the value written
by the other.

So in practice, mixins in this collection must be designed to have relatively
isolated effects. Experience suggests that that *is* possible at the scale of
this project. That is, when designing a collection of components, each of
which is built from 5–20 mixins, it's possible to write mixins such that
their effects are generally independent, and can be applied in any order.

A prototype component collection based on functional mixins has revealed a small
number of cases where it is difficult to avoid ordering mixin method invocation.
See "Drawbacks" and "Unresolved questions".

## Waiting for all prototypes to finish method invocation

Consider a mixin has a `foo` method, and would like to ensure that work will
happen after *all* prototypes along the chain have performed their `foo` work.

    const MyMixin = (base) => class MyMixin extends base {
      foo() {
        if (super.foo) { super.foo(); }
        // Do work here?
      }
    };

Here, MyMixin can guarantee that all base classes further up the prototype chain
have finished doing their `foo` work, because MyMixin explicitly calls
`super.foo()`. However, prototypes further *down* on the prototype chain — that
is, mixins applied after MyMixin, or subclasses otherwise extending this chain —
may still have `foo` work to perform.

    const MyMixin = (base) => class MyMixin extends base {
      foo() {
        if (super.foo) { super.foo(); }
        Promise.resolve().then(() =>
          // All prototypes on the chain will have finished executing their
          // synchronous foo implementations by this point.
        );
      }
    };


# Drawbacks

The relatively rare mixin ordering challenges described above are one drawback
to this approach. Those ordering challenges could be mediated by recourse to an
outside class factory that can try to resolve orderings. However, requiring use
of such a class factor that would be a significant loss. It would impose a
constraint on users of the mixins, and would effectively constituted a new
component framework. This would undoubtedly make the project's mixins less
attractive to a developer already using a component framework.


# Alternatives

This proposal was arrived at over the course of a year of experimentation
looking for alternatives to monolithic UI component frameworks, in which a
comparatively large base class delivers a wide range of component features. We
believe that approach induces significant drag on the evolution of the
framework, as future features inevitably run up against tightly-imposed
constraints in property/method semantics, the ordering of effects, etc.

Another approach would be to deliver all component services as helper functions
that must manually invoked, rather than as extensions to a class. E.g., instead
of offering a mixin to manage the semantics of single selection, each component
that wanted to offer single selection would separate define the relevant API
members, than manually invoke corresponding helper functions. Such an
approach has some appeal, as it gives the component developer complete freedom
to decide when and how to invoke the helper functions. On the downside, this
leads to a more extreme degree of boilerplate code, and makes it harder to
guarantee API consistency.


# Unresolved questions

* Many mixins will want to depend on the existence of a Shadow DOM subtree,
  and there is no standard for the timing of when the tree will be present.
  Some components call `attachShadowRoot` in their constructor, but Polymer 2.0
  components do that at a later point in time. That makes it hard to write
  mixins that inspect or manipulate Shadow DOM. One solution would be for
  the community to agree on the creation of a new *de facto* lifecycle callback
  called, e.g., `shadowRootAttached`.

* We're still exploring
