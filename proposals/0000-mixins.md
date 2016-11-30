- Start Date: 2016-11-17
- RFC PR: (leave this empty)
- Elix Issues: (leave this empty)


# Summary

This proposes that the project agree upon functional mixins as our standard way
to share aspects of behavior and public APIs across our components. Functional
mixins take the form of a plain JavaScript function that takes a class argument
and returns an extended class implementing the mixin's functionality:

    const GreetMixin = (base) => class Greet extends base {
      // Mixin defines properties and methods here.
      greet() {
        return 'Hello';
      }
    };

A mixin is applied simply by invoking the mixin as a function, typically
in the process of defining a class. In the case of mixins for web components,
the base class could be HTMLElement:

    class GreetElement extends GreetMixin(HTMLElement) {}
    let obj = new MyElement();
    obj.greet(); // "Hello"


# Motivation and goals

Web components often share aspects of behavior and public API. For example, many
components support the notion of single selection, where the user can pick at
most one item from the component's children. It is advantageous to implement
such behavior in a reusable fashion at the level below that of a complete
component. This allows for implementation to be shared, and for developers to
interact with a more consistent component API.

Many UI component frameworks support some form of mixin, but differences in
implementation can increase learning costs and hinder reuse. Functional mixins
provide a greater degree of interoperability. They directly leverage the
JavaScript prototype chain, and so can be used in a variety of ways and with a
variety of base classes.

Design goals:

1. **Allow complex components to be cleanly factored into reusable mixins.**
   Each mixin can focus on a single, common component task. Each mixin should be
   useful on its own, and contribute a well-defined aspect of behavior to a
   component. In combination, mixins should be able to define most or all of a
   component's behavior.
2. **Introduce as few new concepts as possible.**
   A developer who understands JavaScript and the DOM API should be able to work
   with Elix mixins without having to substantially change the way they write
   code. They shouldn't have to learn anything new beyond the concept of
   defining a mixin as a function that extends a class.
3. **Assume native browser support for ES6 and web components.**
   The architecture should feel correct in a future world in which native ES6
   and web components are everywhere, but also be usable in older ES5 browsers
   and with polyfills.

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
   mixins to provide multiple implementations of properties and methods with the
   same name.
4. Naming conflicts can be further avoided by using property symbols, not string
   names, to define properties and methods that do not need to be exposed in the
   public API.

Non-goals:

While this mixin strategy is sufficiently general to be of interest to problem
domains outside of web component creation, this plan is focused on the needs of
creating web components in the context of the Elix project. Theoretical
weaknesses in this mixin strategy may be acceptable if they are not likely to
come into play in creating web components like those in the Elix project.

Desired outcomes:

* The creation of a well-factored collection of mixins for all behavior aspects
  shared across Elix project components. Most of the components will be
  primarily comprised of mixins, with very little code required to define the
  final, instantiable custom element.
* A developer that wants to use an Elix component, but does *not* want some
  aspect of its behavior, can create a new component using the same set of
  mixins as the original, minus the mixin(s) they want to leave out. This
  provides a critical degree of flexibility for Elix users.
* The same mixins can be used by developers on other projects to help them
  create proprietary components which also meet the Gold Standard. They can use
  the mixins in vanilla JavaScript, or with any framework that permits the
  creation of web components in JavaScript using `class` syntax. The mixins
  enable a pay-as-you-go approach to web component complexity and performance.


# Use cases

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
* ARIA list semantics. A component that supports single selection (above) would
  like to go further and expose the currently selected item via ARIA attributes
  to make the component accessible to users of, e.g., screen readers.

These mixins can be applied directly to the standard `HTMLElement` class if a
developer wants to work directly on top of the platform. These mixins are also
designed to be used with base classes from web component frameworks.

Specifically, this functional mixin approach is consistent with that being
adopted by Google's Polymer team in Polymer 2.0 as a replacement for Polymer 1.0
component behaviors. Elix mixins can be applied to the `Polymer.Element` base
class to easily define the behavior of Polymer components.


# Detailed design

## Mixins as functions

The mixins in this project all take the form of a function. Each function takes
a base class and returns a subclass defining the desired features.

Unlike many historical JavaScript mixin implementations, functional mixins do
*not* destructively modify a class prototype. Rather, functional mixins extend
the prototype chain and return a new class. Such functions have been called
"higher-order components", but we prefer the term "mixin" for brevity.

The mixins in this project take care to ensure that base class properties and
methods are not broken by the mixin. In particular, if a mixin wants to add a
new property or method, it also invokes the base class' property or method. To
do that consistently, these mixins follow standardized composition rules
(explained in detail further below). Developers interested in creating their own
component mixins may find it helpful to follow those guidelines to ensure that
their mixins can interoperate cleanly with Elix mixins.

A core virtue of a functional mixin is that you do not need to use any library
to apply it. They're just functions, and can be applied just by invoking them.
This lets developers use these mixins with any conventional means of defining
JavaScript classes — they don't have to invoke a proprietary class factory, nor
do they have to load a separate framework or runtime.

Because mixins define behavior through composition, they do not impose the
limitations of a single-inheritance class hierarchy. That said, a developer can
still use them within a class hierarchy if that's suitable for their
application. For example, one can compose a set of mixins to create a custom
base class from which other classes derive. But the use of such a base class is
not required.


## Mixin conventions

### Naming

Mixins in this project are named with an initial capital: `SampleMixin`, not
`sampleMixin`. The initial capital is intended to suggest their status as
partial classes.

The mixin name should generally end in "Mixin". By convention, the mixin's name
without the "Mixin" part should be used as the name of the returned class:

    const SampleMixin = (base) => class Sample extends base { ... }

Here, `SampleMixin` is the visible name of the function that can be applied to a
class to get back an extended class. The class name `Sample` is actually not
required by JavaScript, but is helpful for debugging. It allows the debugger to
show you a meaningful name when you inspect a component's prototype chain at
runtime.


### String names vs `Symbol` keys

Elix mixins use string names for properties and methods which are properly part
of a component's public API. Properties and methods which are only meant as an
internal means for communication between a component and its mixins are
identified with `Symbol` keys.

    // This symbol must be exported so other mixins can see it.
    export const internalMethodSymbol = Symbol('internalMethod');

    export const SampleMixin = (base) => class Sample extends base {

      // Method exposed in a component's public API.
      publicMethod() {
        ...
      }

      // Method only invoked by the component or other mixins.
      [internalMethodSymbol]() {
        ...
      }

    };


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

To illustrate a typical separation of concerns using mixins, let's consider a
component that wishes to let the user select a child element by clicking on it.
Rather than writing this in a monolithic fashion, where the component directly
handles all aspects of this situation, the component employs three mixins.

1. A ClickSelectionMixin that wires up a click handler to listen for clicks on the
   component's children. When a child is clicked, the mixin sets a property with
   the standardized name `selectedItem` to the clicked child.
2. A SingleSelectionMixin that tracks a single selected item. It defines a
   `selectedItem` property whose setter saves the selected element, and a getter
   that can be used later to retrieve that element. The `selectedItem` setter
   also invokes a method `applySelection` for any item that becomes selected or
   deselected.
3. A SelectionClassMixin that toggles a `selected` CSS class on elements that
   become selected or deselected. It does this by
   item. It does this by defining an implementation of the `applySelection`
   method that applies the `selected` class to an individual item. The actual
   visible effects of the `selected` class can then be defined via CSS rules.

Please see this complete [Sample Mixin-Based
Component](http://jsbin.com/wikowa/edit?html,output) for a functioning demo.
The general structure of the sample is as follows:

    // Define three mixins.
    const ClickSelectionMixin = base => class ClickSelection extends base {...}
    const SingleSelectionMixin = base => class SingleSelection extends base {...}
    const SelectionClassMixin = base => class SelectionClass extends base {...}

    // Apply the mixins.
    class TestElement extends
        ClickSelectionMixin(SelectionClassMixin(SingleSelectionMixin(HTMLElement))) {}
    customElements.define('test-element', TestElement);

Each mixin performs a single task. When all three are applied to a component
class, the three work together to give the component basic selection behavior.
The user can click a child element of the component and that selected element
will be highlighted via whatever styling the application deems appropriate.
Programmatically, the application can set or get the component's selected child
via its public `selectedItem` property.

Such a factoring may initially feel overly complex, but has some key advantages:

* Each mixin is focused on doing just one job well. Even though these jobs
  sound simple, in the real-world each actually ends up handling considerable
  complexity.
* The loose arrangement permits a substantial degree of developer freedom. A
  developer who decides they want to handle clicks differently can replace
  ClickSelectionMixin with code of their own, and still take advantage of the
  other two mixins. They could change the rendering of the selection by
  replacing SelectionClassMixin. Or they can keep those mixins and *add* others
  to handle other forms of input or other rendering effects (e.g., ARIA
  support).
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

However, mixins may be applied to a variety of base classes, including classes
returned by other mixins. Mixin authors should generally write methods and
properties that augment base class functionality rather than replacing it. This
increases the chance that base class functionality will continue to work even in
the presence of mixins.

Generally, this means that mixin methods and properties should invoke `super`.
Unlike the normal application of `super` in subclasses, mixins do not have any
guarantee that a base class actually implements a given method or property.
Hence, the mixin cannot just blindly invoke `super` — it must generally inspect
the base class first for the presence of the method or property and, only if it
exists, invoke `super`.

There is no single solution that works for all kinds of mixin members. Instead,
you can follow the series of rules presented below.


### Extending a method known to not return a result

This is the simplest case. Your mixin method should check to see whether the
base defines a method of the same name, and if so, invoke that method. In most
cases, you should invoke the super method before performing your own mixin work.

    const SampleMixin = (base) => class Sample extends base {
      method(...args) {
        if (super.method) { super.method(...args); }
        // Do your mixin work here.
      }
    };

Be sure to pass along any arguments to the base class’ method implementation.


### Extending a method that may return a result

If the base class method might return a result, you should capture that result
before doing mixin work. Your mixin has the opportunity to modify the base
method’s result, or can leave it unchanged. In any event, your mixin method
should itself return a result.

    const SampleMixin = (base) => class Sample extends base {
      method(...args) {
        let result = super.method && super.method(...args);
        // Do your mixin work here, potentially modifying result.
        return result;
      }
    };


### Extending a property getter with no setter

If you’re certain that a property will *never* have a setter, you can omit
defining a setter. But if any other mixin might define a setter for the
property, you need to define one, too. Otherwise the absence of a setter will
implicitly override a setter further up the prototype chain.

To avoid this problem, define a default setter that checks to see whether it
should invoke super. To check the base class for the existence of a property,
use the idiom, `'name' in base.prototype`.

If your getter wants to override the base class implementation, it does not need
to invoke super. Otherwise, it can invoke the super getter, then perform
modifications to that result before returning it.

    const SampleMixin = (base) => class Sample extends base {
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


### Extending a property setter with no getter

It’s possible that your mixin will want to define a setter but not a getter:
your mixin may want to do work when the property changes, but leave the actual
retrieval of the property to a base class.

In such situations, you must still supply a default getter that invokes super.
(Otherwise, the absence of a getter in your mixin will implicitly override a
getter defined further up the prototype chain.) Your getter does not need to
check to see whether it should invoke super — if the base class doesn’t define
the property, the result will be undefined anyway.

    const SampleMixin = (base) => class Sample extends base {
      get property() {
        return super.property;
      }
      set property(value) {
        if ('property' in base.prototype) { super.property = value; }
        // Do your mixin work here.
      }
    };


### Extending a property with both getter and setter

This is a combination of the above two rules. The getter will generally want to
override the base class property, so it doesn’t need to invoke super. The setter
should use the same technique above to see whether it should invoke super.

    const SampleMixin = (base) => class Sample extends base {
      get property() {
        // Do your mixin work here.
      }
      set property(value) {
        if ('property' in base.prototype) { super.property = value; }
        // Do your mixin work here.
      }
    };


### Extending a property with a getter and setter that "backs" a value

A common case of the above rule is a mixin that will store the value of
property for the benefit of other mixins and classes. Such a mixin is said to
"back" the property.

In this situation, the recommendation is to have the setter record the new value
of the property *before* invoking `super`. This ensures that, if the base class'
property setter immediately inspects the property's value via the getter, the
latest value will be returned. All other work of the mixin should be done
*after* invoking `super`, as usual.

    let propertySymbol = Symbol('property');

    const SampleMixin = (base) => class Sample extends base {
      get property() {
        return this[propertySymbol];
      }
      set property(value) {
        // Save latest value before invoking super.
        this[propertySymbol] = value;
        // At this point, getter invocations will get the right value.
        // Now invoke super.
        if ('property' in base.prototype) { super.property = value; }
        // Do any other mixin work here.
      }
    };


## Mixin order independence

A critical goal is designing web component mixins which can be applied in any
order. That is, if `FooMixin()` and `BarMixin()` are functional mixins, then
both `FooMixin(BarMixin(HTMLElement))` and `BarMixin(FooMixin(HTMLElement))`
should return classes whose prototype chains are slightly different, but whose
behavior is otherwise the same. Order independence makes it much easier for
developers to use mixins.

The general nature of this mixin architecture does not guarantee order
independence: two people could easily write two mixins that modify the same
element attribute, in which case one mixin might overwrite the value written
by the other. That means Elix mixins must be designed to have relatively
isolated effects.

Experience suggests that is possible at the scale of this project. That is, when
designing a collection of components, each of which is built from 5–20 mixins,
it's possible to write mixins such that their effects are generally independent,
and can be applied in any order.

A prototype component collection based on functional mixins has revealed a small
number of cases where it is difficult to avoid ordering mixin method invocation.
(See "Drawbacks".)


## Waiting for all prototypes to finish method invocation

Consider a mixin that has a `foo` method, and would like to ensure that work
will happen after *all* prototypes along the chain have performed their `foo`
work.

    const SampleMixin = (base) => class Sample extends base {
      foo() {
        if (super.foo) { super.foo(); }
        // Do work here?
      }
    };

Here, SampleMixin can guarantee that all base classes further *up* the prototype
chain have finished doing their `foo` work, because SampleMixin explicitly calls
`super.foo()`. However, prototypes further *down* on the prototype chain — that
is, mixins applied after SampleMixin, or subclasses otherwise extending this
chain — may still have `foo` work to perform.

It's our experience that this situation does not come up often in practice. When
it has, we have found that the simplest solution may to be enqueue a microtask:

    const SampleMixin = (base) => class Sample extends base {
      foo() {
        if (super.foo) { super.foo(); }
        Promise.resolve().then(() =>
          // All prototypes up and down the prototype chain will have finished
          // executing their synchronous `foo` implementations by this point.
        );
      }
    };

This has the advantage of simplicity, but gives up synchronicity.


## Performance

Our explorations suggest that Elix components might end up with 5–20 mixins
each. Those components would have deeper JavaScript prototype chains than is
typically encountered in traditional class hierarchies. Some people have
expressed concerns about the performance impact of such deep prototype chains.

We care a great deal about performance, so the topic deserves careful study. But
initial testing suggests that the performance impact of a deep prototype chain
on its own may be negligible, and in any event comparable to that of other means
of composing component behavior from smaller parts.

Early in our explorations, we conducted a [simple peformance
comparison](https://github.com/ComponentKitchen/wc-perf) of Polymer 1.0
components with 20 behaviors against a plain JavaScript component with
20 mixins. The results described there suggest that this mixin approach
does not incur a significant performance penalty just by virtue of creating
long prototype chains.

It stands to reason that, as the prototype chain is a core part of how
JavaScript works, the creators of JavaScript engines have worked hard to
make property and method invocation along the chain as fast as possible.

That's not to say that all mixins are automatically fast. A poorly-written mixin
could, of course, have serious performance problems for other reasons. So
performance testing will be critical to creating high-quality mixins and
components for Elix.


# Drawbacks

The relatively rare mixin ordering challenges described above are one drawback
to this approach. One case we have encountered in practice is having one mixin
whose constructor creates a shadow root, and another mixin whose constructor
wants to presume the existence of a shadow root. In this situation, the first
mixin must be applied before the second. (This specific issue also pertains to
using Elix mixins with Polymer, and is captured below in "Unresolved
questions".)

Such ordering challenges could be mediated by recourse to an outside class
factory that can try to resolve orderings. However, requiring use of such a
class factory would be a significant loss. It would impose a constraint on users
of the mixins, and would effectively constitute a new component framework. This
would undoubtedly make the project's mixins less attractive to a developer
already using a component framework.

In practice, component mixins will often need to directly manage component
state. Functional-Reactive Programming approaches such as React take a very
different view of state. While Elix _components_ are intended to be useful in
FRP contexts like React as packaged implementations of user interface patterns,
Elix _mixins_ on their own are not designed to be applied directly to React
component classes.

As yet, we have not designed a means to ensure that a given mixin is only
applied once along a given prototype chain. It is up to the component creator to
ensure they don't inadvertently apply a mixin twice. That might not seem like a
large problem, but it does limit the ability to compose aggregate mixins from
smaller mixins. In practice, this generally means that the component create
applies all mixins directly in the creation of the component class.


# Alternatives

This proposal was arrived at over the course of a year of experimentation
looking for alternatives to monolithic UI component frameworks, in which a
comparatively large base class delivers a wide range of component features. We
believe a monolithic approach induces significant drag on the evolution of the
framework, as future features inevitably run up against tightly-imposed
constraints in property/method semantics, the ordering of effects, etc.

The proposal section on "Composition rules for extending base class methods and
properties in a mixin" generates a degree of boilerplate code in mixins.
Alternatives were explored to reduce this boilerplate, either through the use of
helper functions or declarative property/method definition. However, while those
approaches allow for shorter code, they layer on abstraction and so lose
clarity. A developer would have to learn those abstractions to understand how
the mixins did their work. In a project that's intended to draw contributions
from a broad audience of developers, many of whom will use Elix components in
the context of some other front-end framework, we feel that it's better to have
plainer code that can speak for itself.

An alternative to the mixin approach laid out here would be to deliver all
component services as helper functions that must be invoked manually, rather
than as extensions to a class. E.g., instead of offering a mixin to manage the
semantics of single selection, each component that wanted to offer single
selection would separately define the relevant API members, than manually invoke
corresponding helper functions. Such an approach has some appeal, as it gives
the component developer complete freedom to decide when and how to invoke the
helper functions. On the downside, this leads to a more extreme degree of
boilerplate code, such that it becomes much harder to write components and
harder to see what a component is actually doing. It also makes it harder to
guarantee API consistency, as each component must manually expose every member
of its public API.


# Unresolved questions

* Many component mixins want to wait for the creation of a Shadow DOM subtree,
  and there is no standard for the timing of when the tree will be present.
  Some components call `attachShadowRoot` in their constructor, but Polymer 2.0
  components do that at a later point in time. That makes it hard to write
  mixins that inspect or manipulate Shadow DOM during component creation. One
  solution would be for the community to agree on the creation of a new *de
  facto* lifecycle callback called, e.g., `shadowRootAttached`.
