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
component. This allows for implementation to be shared, and for developers to
interact with a more consistent component API.

Many UI component frameworks mixins, but differences in implementation can
increase  learning costs and hinder reuse. Functional mixins provide a greater
degree of interoperability. They directly leverage the JavaScript prototype
chain, and so can be used in a variety of ways, and with a variety of base
classes.

Design goals:

1. **Allow mixins that can focus on a single, common component task.**
   Each mixin should be useful on its own, or in combination.
2. **Introduce as few new concepts as possible.**
   A developer who understands the DOM API should be able to work with these
   mixins without having to substantially change the way they write code. They
   shouldn't have to learn anything new beyond the concept of defining a mixin
   as a function.
3. **Assume native browser support for ES6 and web components.**
   The architecture should feel correct in a future world in which native ES6
   and web components are everywhere, but also be usable in older ES5 browsers
   and with polyfills.

Use cases for functional mixins for web components:

* Template stamping. Components often want to create a new shadow root in their
  constructor and clone a template into it. Since this behavior is fairly simple
  and consistent, the developer would like a mixin to handle this.
* Attribute marshalling. A component would like to implement a default
  `attributeChangedCallback` to marshall hyphenated `foo-bar` attributes to
  the corresponding camelCase `fooBar` properties.
* Single selection. A component would like to implement standard single-selection
  semantics, with a `selectedItem` property, `selectNext`/`selectPrevious`
  methods, a `selected-item-changed` event, and so on.
* ARIA list semantics. A component that supports selection (above) would like to
  go further and expose the currently selected item via ARIA attributes to make
  the component accessible to users of, e.g., screen readers.

These mixins can be applied directly to the standard `HTMLElement` class if a
developer wants to work directly on top of the platform. These mixins are also
designed to be used with base classes from web component frameworks.
Significantly, this functional mixin approach is consistent with that adopted by
Google's Polymer team for their future releases. Polymer's earlier, proprietary
"behaviors" is being replaced with mixins such as those described here.

This mixin strategy is designed to mitigate common problems with earlier mixin
approaches, for example the mixin approach which is now  
[deprecated in React](https://facebook.github.io/react/blog/2016/07/13/mixins-considered-harmful.html).
The mixin strategy described here avoids a number of the problems with classical
mixins:

1. The functional mixins described here are closer in spirit to what React calls
   "higher-order components" than traditional JavaScript mixins, which typically
   destructively modify an object prototype.
2. Mixins are only used to provide behavior that affects or interacts with the
   component's public API, or hooks into web component lifecycle methods.
3. Naming conflicts are resolved using the JavaScript prototype chain. A
   mixin can invoke base class functionality via `super`, allowing multiple
   mixins can be used together.
4. Naming conflicts are further avoided by using property symbols, not string
   names, to access properties and methods that do not need to be exposed in the
   public API.

While this mixin strategy is sufficiently general to be of interest to problem
domains outside of web component creation, this plan is focused on the needs of
creating web components in the context of the Elix project. Theoretical
weaknesses in this mixin strategy may be acceptable if they are not likely to
come into play in creating Elix components.

The expected outcome of this design:

* The creation of a well-factored collection of mixins for all behavior aspects
  shared across Elix project components. Most of the components will be
  primarily comprised of mixins, with very little code required to define the
  final, instantiable custom element.
* A developer that wants to use an Elix component, but does *not* want some
  aspect of its behavior, can create a new component that uses the same set of
  mixins as the original, minus the mixin(s) they want to leave out. This
  provides a critical degree of flexibility for Elix users.
* The same mixins can be used by developers on other projects to help them
  create components which meet the Gold Standard. They can use the mixins in
  "plain JavaScript", or with  any framework that permits the creation of web
  components in JavaScript using `class` syntax. The mixins permit a
  pay-as-you-go approach to web component complexity and performance.


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
(below). Developers interested in creating their own component mixins may find
it helpful to follow those guidelines to ensure that their mixins can
interoperate cleanly with the ones in this project.

A core virtue of a functional mixin is that you do not need to use any library
to apply it. They're just functions, and be applied just by invoking them. This
lets developers use these mixins with any conventional means of defining
JavaScript classes — they don't have to invoke a proprietary class factory, nor
do they have to load a separate framework or runtime.

Because mixins define behavior through composition, they do not impose the
limitations of a single-inheritance class hierarchy. That said, a developer can
still use them within a class hierarchy if that's suitable for their
application. For example, one can compose a set of mixins to create a custom
base class from which other classes derive. But the use of such a base class is
not required.


## Mixin conventions

Mixins in this project are named with an initial capital: `MyMixin`, and not
`myMixin`. The initial capital is intended to suggest their status as
partial quasi-classes.

The mixin name will generally appears twice in mixin definitions, such as that
shown earlier:

    const MyMixin = (base) => class MyMixin extends base { ... }

The first use of `MyMixin` is the visible name of the function that can be
applied to a class to get back an extended class. The second `MyMixin` is
actually not required, but is helpful for debugging. It allows the debugger to
show you a meaningful name when you inspect a component's prototype chain at
runtime.


## Semantic mixin factoring

A core intention behind the use of mixins for this project is to factor
component services into separable but complementary mixins. This loose
arrangement permits a close correspondance between behaviors and code modules,
which makes the code easier to debug and maintain. It also allows mixins to
be flexibly recombined in new ways for new situations.

Web component mixins will generally fall into three categories: 1) mixins that
deal with user input (via event handlers) or other external factors, 2) mixins
that perform purely internal work, often mapping from one level of abstraction
to another, and 3) mixins that deal with rendering (via DOM manipulations) and
other forms of output.

Mixins can communicate with each other and with the component they are part of
via properties and methods that have standardized string names or shared
`Symbol` identifiers.

For example, suppose a component wishes to let the user select a child element
by clicking on it. Rather than having the component directly handle all
aspects of this situation, the component employs three mixins:

1. An input mixin wires up a click handler to listen for clicks on the
   component's children. When a child is clicked, the mixin sets a property with
   the standardized name `selectedItem` to the clicked child. This mixin
   provides a baseline implementation of the property which does nothing.
2. An abstract mixin tracks single selection. It defines a `selectedItem`
   property whose setter saves the selected element, and a getter that can be
   used later to retrieve that element.
3. A rendering mixin takes care of applying a `selected` CSS class to a
   selected element. It does this by defining a getter/setter for the same
   `selectedItem` property. When `selectedItem` is set, the mixin removes the
   `selected` class from any previously-selected element, and applies the class
   to the newly-selected element. The actual visible effects of the `selected`
   class can be defined via CSS.

Each mixin performs a simple task, but when all three are applied to a component
class, the three work together to give the component basic selection behavior.
The user can click a child element of the component and that selected element
will be highlighted via whatever styling the application deems appropriate. The
application can set or get the component's selected child via its public
`selectedItem` property.

All three of these mixins define a `selectedItem` property, but because they
do so using standard composition rules (below), all three property
implementations are mutually compatible. A developer can apply the mixins to
a base class (HTMLElement, say) in any order and get the same result.

Such factoring may initially feel overly complex, but has some key advantages:

* Each mixin is focused on doing just one job well. Even though these jobs
  sound simple, in the real-world each actually ends up handling considerable
  complexity.
* The loose arrangement permits a substantial degree of developer freedom. A
  developer might decide they want to handle clicks differently, or handle
  other forms of input as well. They may also want to render selection
  differently, or have other effects happen when a new element becomes selected.
* In general, this factoring allows components with radically different
  interaction models or visual presentations to nevertheless share a
  considerable amount of user interface logic.


## Composition rules for extending base class methods and properties in a mixin

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

This mixin approach is generally predicated on components that maintain state.
Functional-Reactive Programming approaches such as React take a very different
view of state. While Elix components are intended to be useful in FRP contexts
like React as packaged implementations of user interface patterns, Elix mixins
on their own are not designed to be applied to React component classes.


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

* The section above on "Waiting for all prototypes to finish method invocation"
  makes recourse to microtasks as a solution. This feels somewhat unsatisfying.
