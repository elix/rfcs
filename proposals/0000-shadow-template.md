- Start Date: 2017-01-12
- RFC PR: https://github.com/elix/rfcs/pull/5
- Elix Issues: (leave this empty)


# Summary

This proposes a mixin called `ShadowTemplateMixin` that helps plain JavaScript
components stamp a template into a new shadow root during instantiation.

    class GreetElement extends ShadowTemplate(HTMLElement) {
      get [symbols.template]() {
        return `
          Hello, <slot></slot>!
        `;
      }
    }

    const element = new GreetElement();
    element.textContent = 'world';      // User sees, "Hello, world!"

When the element above is instantiated, the constructor supplied by
`ShadowTemplateMixin` finds the element's template and stamps it into a new
shadow root. It also ensures the template will work with the ShadyCSS polyfill
if that's being used.


# Motivation

Defining a template and stamping it into a shadow root is a core web
component task. The task entails enough complexity that a reusable solution
is very helpful. `ShadowTemplateMixin` seeks to solve that need as simply as
possible.

Desired outcomes:

1. Plain JavaScript components have an easy, flexible way to define a template
   and populate a shadow root, and do so in a way that works with the polyfills.

2. Elix components can share a consistent means of defining a template that does
   not require a full component framework.

3. Isolate the means by which a shadow root is populated to a single mixin so
   that other Elix mixins can interoperate cleanly with shadow roots populated
   by other frameworks (e.g., Polymer).

4. Avoid a dependency on HTML Imports, a spec which has never received full
   approval, and which is still in flux. At the same time, allow Elix components
   to be defined through HTML Imports for teams that prefer them.


Non-goals:

* This mixin is not intended to be the only means by which component developers
  populate a shadow root. Most Elix mixins are designed to work with a shadow
  root constructed by any means. In situations where `ShadowTemplateMixin` does
  not meet a developer's needs, they have complete freedom to replace it.


# Use cases

The `ShadowTemplateMixin` is a fundamental bit of code that will likely find
applicability in all Elix components that have a shadow root – that is, nearly
all of the project's components.


# Detailed design

The mixin does the following:

* In a component's `constructor`, the mixin looks for a template property
  (below).
* If the ShadyCSS polyfill is loaded, the mixin uses it to prepare the template.
* The mixin attaches a new, open shadow root.
* The mixin clones the prepared template into the shadow root.
* If the ShadyCSS polyfill is loaded, in the component's `connectedCallback`,
  the mixin invokes ShadyCSS to apply styles to the new component instance.


## The template property

The `ShadowTemplateMixin` expects a component to define a property getter
identified as `[symbols.template]`. If this property has not been defined, the
mixin issues a console warning, but does not throw an exception.

The property can return a component template as either a real `<template>`
element or, for convenience, as a string. The mixin will upgrade a string
template to a real `<template>`.

The template can come from anywhere. It can be embedded in the JavaScript as a
backquoted template string (as shown in the Summary example), or it be loaded
from some other resource. E.g., if HTML Imports are used, the template could be
obtained from the same HTML file in which the class is defined:

    <template>
      Hello, <slot></slot>!
    </template>

    <script>
      const template = document.currentScript.ownerDocument.querySelector('template');
      class GreetElement extends ShadowTemplate(HTMLElement) {
        get [symbols.template]() {
          return template;
        }
      }
    </script>

_Note: The above code is theoretical, as it does not answer the question of how
to perform a JavaScript `import` in an HTML Import._


## Template caching

For better performance, `ShadowTemplateMixin` caches a component's processed
template.

The mixin reads the `[symbols.template]` property once, in the `constructor`.
After processing the template (upgrading a string template to a `<template>`,
and preparing the template for use with ShadyCSS), the processed template is
cached.

Subsequent component instantiations will use the cached template directly.


# Drawbacks

`ShadowTemplateMixin` represents the beginnings of a new component core for Elix
components – most Elix components will depend upon it. By design, it's not a
piece required by the other mixins, and a template can be stamped in other ways,
but this mixin nevertheless lays a foundation that acts like a component base
class. That's a strength, but also represents a learning curve for developers
wishing to use Elix components.


# Alternatives

In theory, we could have Elix components do their own template stamping, without
relying on a mixin. However, doing so would be cumbersome, particularly in light
of the work required to support the ShadyCSS polyfill.

Elix could also decide to adopt another project's element base class, and rely
on it for basic component services like template stamping. The most likely
candidate is Polymer's `Polymer.Element`. We could use that as an element base
class or, since `Polymer.Element` is itself constructed from mixins, we could
consider using its [template-stamping
mixin](https://github.com/Polymer/polymer/blob/2.0-preview/src/template/template-stamp.html).

However, using `Polymer.Element` or its mixins would introduce overhead (the
template-stamping mixin does work to support data binding, which we may not
need), undesirable entanglements (including mandatory use of HTML Imports), and
heavy conceptual load for a plain JavaScript component developer. Given the
relatively straightforward requirements Elix has for template stamping, it seems
preferable to handle that task ourselves.
