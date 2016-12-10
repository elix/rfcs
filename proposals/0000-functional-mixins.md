- Start Date: 2016-11-17
- RFC PR: https://github.com/elix/rfcs/pull/1
- Elix Issues: (leave this empty)


# Summary

This proposes that the Elix project agree upon functional mixins as a standard
way to share aspects of behavior and public APIs across our components.
Functional mixins take the form of a plain JavaScript function that accepts a
class argument and returns an extended class adding the mixin's functionality:

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


# Motivation

Web components often share aspects of behavior and public API. For example, many
components support the notion of single selection, where the user can pick at
most one item from the component's children. It is advantageous to implement
such common behavior in a reusable fashion at the level below that of a complete
component. This allows for implementation to be shared, and for developers to
interact with a more consistent component API.

Many UI component frameworks support some form of mixin, but differences in
proprietary mixin strategies increase learning costs and hinder reuse. Specific
mixin solutions can also suffer from complex, subtle, or unintended conflicts
between mixins. Functional mixins, as described here, should provide a greater
degree of interoperability with fewer negative interactions, and hence better
scalability.

Desired outcomes:

1. **A high degree of code sharing between Elix components.**
   Essentially all common aspects of component behavior exhibited by multiple
   Elix components will be packaged as reusable mixins. Many components will be
   primarily comprised of mixins, with very little code required to define the
   final, instantiable custom element.
2. **Mixins closely correspond to well-defined aspects of component behavior.**
   Each mixin will focus on a single, clear task. They should be usable on their
   own, and work well in combination with other mixins. It should be easy for
   Elix contributors tracking the origin of a given component behavior to find
   and modify the appropriate mixin providing the behavior.
3. **It is comparatively easy for new contributors to understand Elix code.**
   This mixin architecture introduces as few new concepts as possible. A
   developer who understands JavaScript and the DOM API should be able to work
   with Elix mixins without having to substantially change the way they write
   code. They shouldn't have to learn anything new beyond the concept of
   defining a mixin as a function that extends a class, and some sensible
   conventions for defining mixin properties and methods.
4. **A pay-as-you-go approach to web component complexity and performance.**
   In situations where components do *not* need to share behavior, it should be
   easy to leave behavior out by leaving out the corresponding mixins. Creating
   a variation on an existing component should be a matter of duplicating the
   existing component's list of mixins and omitting the ones which are not
   necessary.
5. **Other developers can use Elix mixins to create their own components.**
   The same mixins can be used by developers on other projects to help them
   create proprietary components which also meet the Gold Standard. They can use
   the mixins in vanilla JavaScript, or with any framework that permits the
   creation of web components in JavaScript using `class` syntax. As noted
   above, a developer can create a variation of an existing Elix component that
   leaves out the mixins they don't need or which they want to replace. This
   should provides a critical degree of flexibility, and lets Elix avoid the
   impossible task of satisfying everyone's diverse needs.
6. **A smooth migration from ES5 to ES6+.**
   This architecture should feel correct in a future world in which native ES6+
   and web components are everywhere, but also be usable today in older ES5
   browsers and with polyfills.

Non-goals:

While this mixin strategy is sufficiently general to be of interest to problem
domains outside of web component creation, this plan is focused squarely on the
needs of creating web components for Elix. Theoretical weaknesses in this mixin
strategy may be acceptable if they are not likely to actually come into play in
writing Elix web components.

In the same vein, this plan is not intended as the only way Elix component
behavior can be defined. As the developer community gains experience creating
large collections of web components, new paradigms will emerge that improve upon
the architecture proposed here. Elix should certainly entertain adopting new
means of fulfilling its mission. This plan is simply good enough for us to get
started. Moreover, creating a well-factored collection of mixins now should make
it easier to migrate to whatever better architectures might come along in the
future.

Mixins are not intended to be the only means by which code is shared across
components.

* Generally speaking, mixins will be used to provide behavior that affects
  or interacts with the component's public API, or hooks into web component
  lifecycle methods.
* In situations other than those listed above, shared helper functions are
  typically a much simpler and more conventional way to reuse code.
* In cases where components should directly share visible user interface
  elements (e.g., two components both want to include the same kind of button),
  packaging the shared elements as a web component is, of course, a good
  strategy for sharing code.


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

All mixin strategies wrestle with how to order mixin property/method effects and
avoid naming conflicts. Common solutions involve destructively modifying a
target object's prototype, or generating property/method wrappers to broker name
mixin property/method invocations. Those solutions can create brittle solutions
which are hard to debug, maintain, or extend.

In contrast, this proposal calls for functional mixins to extend the JavaScript
prototype chain. This has the immediate benefit of leveraging a standard,
performant, and well-documented feature of the JavaScript language.
Additional benefits:

1. Mixin effects on the prototype chain is deterministically established by the order of mixin application.
   As will be discussed later, mixins will generally be designed so that the
   order of application does not matter. The point here is that any property or
   invocations can be reasoned about. As with any JavaScript property/method
   invoked on an object, the JavaScript engine walks up the prototype chain to
   find the first prototype which implements the given property/method. Applying
   two mixins to a base class, e.g., `FooMixin(BarMixin(HTMLElement))` creates a
   prototype chain in which the prototype added by `FooMixin` is inspected
   first, then the prototype added by `BarMixin`, and finally the prototype of
   `HTMLElement` (and `Object`).
2. Any property/method invocation can invoke base class functionality (that is,
   implementations further up the prototype chain) via the standard `super`
   keyword. This allows multiple mixins to cooperatively provide multiple
   implementations of properties and methods with the same name. The invocation
   of `super` is explicit, making the code easier to understand, trace, and step
   through.

To ensure that Elix mixins cooperate consistently via the prototype chain, they
will follow standardized property and method composition rules explained in
detail further below. Developers interested in creating their own components
with Elix mixins will find it helpful to follow those same conventions to ensure
that their code can interoperate cleanly with Elix's.

Because mixins define behavior through composition, they do not impose the
limitations of a single-inheritance class hierarchy. That said, a developer can
still use them within a class hierarchy if that's suitable for their
application. For example, one can compose a set of mixins to create a custom
base class from which other classes derive. But the use of such a base class is
not required.

A core virtue of a functional mixin is that one does not need to use any library
to apply it. They're just functions, and can be applied just by invoking them.
This lets developers use these mixins with any conventional means of defining
JavaScript classes — they don't have to invoke a proprietary class factory, nor
do they have to load a separate framework or runtime.

Overall, experience with the use of functional mixins along the lines described
here suggest that they are sufficiently flexible and powerful to create a large
web component library at Elix's scale.


## Mixin conventions

### Naming

Mixins in this project are named with an initial capital: `SampleMixin`, not
`sampleMixin`. The initial capital is intended to suggest their status as
partial classes.

The mixin name should generally end in "Mixin". By convention, the mixin's name
_without_ the "Mixin" part should be used as the name of the returned class:

    const SampleMixin = (base) => class Sample extends base { ... }

Here, `SampleMixin` is the visible name of the function that can be applied to a
class to get back an extended class. The class name `Sample` is actually not
required by JavaScript, but is helpful for debugging. It allows the debugger to
show you a meaningful name when you inspect the component's prototype chain at
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

The Elix project will designate a standard set of property and method
identifiers to facilitate communication between a mixin and other mixins or the
overall component. As described above, those identifiers will be strings for
public API members and `Symbol` keys for internal members. The designation of
standard identifiers should help further reduce the chance for naming conflicts
if an external developer applies an Elix mixin to their component. The use of
standard string names for public members also helps ensure API consistency.


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
`Symbol` keys. For example, a mixin may invoke a method with the expectation
that the real implementation of that method will be provided by another mixin
or the overall component.

To illustrate a typical separation of concerns using mixins, let's consider a
modestly complex example. Suppose we wish to create a component that lets the
user select a child element by clicking on it. Rather than writing this in a
monolithic fashion, where the component directly handles all aspects of this
situation, we separate the concerns into three mixins:

1. A ClickSelectionMixin that wires up a click handler to listen for clicks on
   the component's children. When a child is clicked, the mixin sets a property
   with the standardized name `selectedItem` to the clicked child.
2. A SingleSelectionMixin that tracks a single selected item. It defines a
   `selectedItem` property whose setter saves the selected element, and a getter
   that can be used later to retrieve that element. The `selectedItem` setter
   also invokes a method `applySelection` for any item that becomes selected or
   deselected.
3. A SelectionClassMixin that toggles a `selected` CSS class on elements that
   become selected or deselected. It does this by item. It does this by defining
   an implementation of the `applySelection` method that applies the `selected`
   CSS class to an individual item. The actual visible effects of the `selected`
   class can then be defined via CSS rules for `.selected`.

Please see this complete [Sample Mixin-Based
Component](http://jsbin.com/wikowa/edit?html,output) for a functioning demo.
The general structure of the sample implementation is as follows:

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
Moreover, the application can programmatically set or get the component's
selected child via its public `selectedItem` property.

Such a factoring may initially feel overly complex, but has some key advantages:

* Each mixin is focused on doing just one job well. Even though these jobs
  sound simple, in the real-world each actually entails considerable complexity.
* The loose arrangement permits a substantial degree of developer freedom. A
  developer who decides they want to handle clicks differently can replace
  ClickSelectionMixin with code of their own, and still take advantage of the
  other two mixins. Or they could change the rendering of the selection by
  replacing SelectionClassMixin. Or they could keep all three mixins and add
  others to handle other forms of input or other rendering effects (e.g., ARIA
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
properties that invoke base class functionality rather than override it. This
increases the chance that base class functionality will continue to work even in
the presence of mixins.

Generally, this means that mixin methods and properties should invoke `super`.
Unlike the normal application of `super` in subclasses, mixins do not have any
guarantee that a base class actually implements a given method or property.
Hence, the mixin cannot just blindly invoke `super`. It must first look further
up the prototype chain for the presence of the method or property before
deciding whether it can invoke `super`.

There is no single solution that works for all kinds of mixin members. Instead,
if you are writing an Elix mixin, you can follow the conventions presented here.


### Extending a method known to not return a result

This is the simplest case. Your mixin method should check to see whether the
base class defines a method of the same name, and if so, invoke that method. In
most cases, you should invoke the super method before performing your own mixin
work.

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
implicitly hide a setter further up the prototype chain.

To avoid this problem, define a default setter that checks to see whether it
should invoke super. To check the base class for the existence of a property,
use the idiom, `'name' in base.prototype`.

If your getter wants to completely override the base class implementation, it
does not need to invoke super.

    const SampleMixin = (base) => class Sample extends base {
      get property() {
        // Do your mixin work here.
      }
      set property(value) {
        if ('property' in base.prototype) { super.property = value; }
      }
    };

Note the use of `base.prototype` instead of `super` in the setter, as `'name' in
super` is not legal ES6. However, the use of `super` *is* required in the
actual `super.prototype = value` assignment which follows. Using
`base.prototype` there wouldn’t work: it would overwrite the base’s setter
instead of invoking it.


### Extending a property setter with no getter

It’s possible that your mixin will want to define a setter but not a getter:
your mixin may want to do work when the property changes, but leave the actual
storage and retrieval of the property to a base class. (See the discussion of
"backed" properties below.)

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

That is not a definitive result, but it seems reasonable. As the prototype
chain is a core part of JavaScript, it stands to reason that JavaScript engines
work hard to make property and method invocation along the chain as fast as
possible.

That's not to say that all mixins are automatically fast. A poorly-written mixin
could, of course, have serious performance problems for other reasons. So
ongoing performance testing will be critical in creating high-quality mixins and
components for Elix.


## Testing

The entire point of mixins is to reuse functionality across components, so it's
to everyone's benefit for mixins to be as well-tested as possible. Each mixin
should have a reasonably complete suite of unit tests to exercise its
functionality.

Mixin unit tests should try to test the mixin in isolation, and avoid including
other mixins or base classes. Typically, that means applying the mixin directly
to `HTMLElement` to create a custom element class that can be used to create
unit test fixtures. Example: suppose a simple mixin looks like:

    const FooMixin = (base) => class Foo extends base {
      get foo() {
        return 'Hello';
      }
    }

The unit tests for this mixin would typically look like:

    class FooMixinElement extends FooMixin(HTMLElement) {}
    const fixture = new FooMixinElement();
    assert.equal(fixture.foo, 'Hello');

In some cases, it may be expedient to use other mixins in constructing test
fixtures, but over time, it is desirable to isolate which mixin each test
exercises.

Components that use multiple mixins can include tests that exercise the
integration of those mixins. These should generally test the end-user visible
interactions. E.g., if the
[Sample Mixin-Based Component](http://jsbin.com/wikowa/edit?html,output)
introduced above were a real component, one of its tests might simulate a click
on a child, and then confirm that the selected class was ultimately applied to
the clicked child. Such tests don't need to exhaustively duplicate everything
tested in the mixin unit tests, but confirm that the component's basic
functionality works as expected.


# Drawbacks

The relatively rare mixin ordering challenges referenced above are one drawback
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
smaller mixins. In practice, this generally means that code must apply all
mixins directly in the creation of the component class.

This mixin strategy does not prevent entanglements among poorly-coupled
dependencies. It is possible to write two mixins which cannot be used together
or, conversely, two mixins so entangled that one cannot be used without the
other. In other words, it is still possible to write bad code. The task of
designing well-factored mixins will still be challenging, but this proposal does
seem to at least make it feasible.


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

The React community promotes the use of
[higher-order components](https://facebook.github.io/react/blog/2016/07/13/mixins-considered-harmful.html)
instead of mixins. This functional mixin proposal shares at least one key
characteristic with React's higher-order components: both use functions
that accept a class and return a new class. In React's case, the new class
_wraps_ an instance of the original class. In this proposal, the new class
_extends_ the original class. In both cases, the function allows more developer
control over what is happening, and the function's effects are made more readily
apparent to someone reading the code.


# Unresolved questions

* Many component mixins want to wait for the creation of a Shadow DOM subtree,
  and there is no standard for the timing of when the tree will be present.
  Some components call `attachShadowRoot` in their constructor, but Polymer 2.0
  components do that at a later point in time. That makes it hard to write
  mixins that inspect or manipulate Shadow DOM during component creation. One
  solution would be for the community to agree on the creation of a new *de
  facto* lifecycle callback called, e.g., `shadowRootAttached`.
