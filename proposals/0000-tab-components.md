- Start Date: 2017-03-14
- RFC PR: (leave this empty)
- Elix Issues: (leave this empty)


# Summary

This proposes a set of components related to tabbed user interfaces. There are
two components which can be readily used without modification or coding:

* `Tabs`. A set of tabbed panels that can be navigated by selecting
  corresponding tab buttons.

* `LabeledTabs`. A `Tabs` instance with simple tab buttons that have text
  labels.

For flexibility, the components above are build up from secondary components:

* `TabStrip`. A row (or column) of tab buttons. Responsible for positioning the
  buttons, handling keyboard navigation, and supporting accessibility.

* `TabStripWrapper`. Adds a `TabStrip` to a base element, wiring the selection
  states of the two together. For example, the `Tabs` component uses
  `TabStripWrapper` to connect a `TabStrip` to a `Modes` instance.

* `LabeledTabButton`. A classic rounded tab button showing a text label for a
  tab panel. This is used internally by `LabeledTabs` for its tab buttons.

This RFC also proposes some new mixins and helpers:

* `renderArrayAsElements`. Helper function for the common case of rendering
  one element for each item in an array.

* `ShadowReferencesMixin`. Exposes a `$` member on a component that can be used
  to reference elements in its shadow subtree, effectively behaving like
  Polymer's `$` feature.


# Motivation

Desired outcomes:

1. Developers can readily constructed tabbed user interfaces that follow a
   conventional style.
2. Developers can easily adapt these components to created tabbed UIs for more
   specialized situations.

Non-goals:

* Allow extensive customization of the tab components via properties. That
  pattern has many variations, and while it is tempting to address a large
  number of them with configurable properties, that generally leads to
  unmanageable complexity.


# Use cases

The `Tabs` component covers the common modal UI pattern in which the user can
directly control which modal state is presented. Tabs are typically used to
allow a UI to offer more controls than can fit in a confined area at a time.
A common use case is Settings or configuration UIs. Tabs may also be used in a
main window to downplay less-commonly used aspects of a UI.

The classic look of a tabbed dialog is addressed with `LabeledTabs`, but the
base `Tabs` class can be used in many other situations. For example, many
mobile applications offer toolbars presenting 3–5 buttons for the app's main
areas. Each toolbar button makes a different set of controls appear — that is,
the app's primary navigation construct is a set of tabs. Such apps therefore
form an important use case for the `Tabs` component.

The `Tabs` and `LabeledTabs` components assume a standard tabbed UI design in
which clicking a tab immediately makes the corresponding tab panel visible. To
manage that visual transition, those components rely internally on an instance
of `Modes`. However, there are use cases in which an app wishes to provide other
visual effects for the transition between panels, such as a sliding animation.
To accommodate those use cases, the `Tabs` components are constructed from
lower-lever parts that can be easily recombined to add tab panel functionality
around components providing more complex visual transitions.


# Detailed design


## Tabs

The `Tabs` component implements a standard tab panel UI pattern. It is nothing
more than a `TabStrip` (below) wrapped around a `Modes` (above): selecting a
tab button in the tab strip tells the Modes component to display the
corresponding panel. The actual work of wrapping a `TabStrip` around another
element is performed by `TabStripWrapper`.

`Tabs` exposes two slots: 1) a `tabButtons` slot  for the buttons that will be
shown in the tab strip, and 2) a default slot for the tab panels. A simple
example showing this structure:

    <elix-tabs>

      <button slot="tabButtons">Home</button>
      <div>Home page</div>

      <button slot="tabButtons">Search</button>
      <div>Search page</div>

      <button slot="tabButtons">Settings</button>
      <div>Settings page</div>

    </elix-tabs>

This will render three tab panels below a tab strip with three corresponding
buttons. The buttons for the `tabButtons` slot could also be grouped together;
all that matters is that they appear in the same order as the corresponding
tab panels.

See `TabStripWrapper` for a description of the properties exposed by `Tabs`.


## LabeledTabs

Sets the default content of the `tabButtons` slot. If content is assigned to
that slot, it will override the default buttons.

Sets the text label of a LabeledTabButton to the `aria-label` attribute of the
corresponding panel.


## TabStrip

A `TabStrip` exposes two properties:

* `tabAlign`: determines where how the tab buttons will be aligned in the strip:
  "start" (typically left), "center", "end" (typically right), or "stretch." The
  values "start" and "end" are used instead of "left" and "right" to accommodate
  pages in right-to-left languages such as Arabic and Hebrew.
* `tabPosition`: determines how the tab strip itself is positioned relative to
  the tab panels it is controlling: "top", "bottom", "left", "right".  

For styling purposes, both of the above properties are reflected to the
tab strip element as `tab-align` and `tab-position` attributes. Additionally,
since a tab button itself will want to know how it is being positioned relative
to the corresponding tab panel, a `TabStrip` also reflects the `tab-position`


## TabStripWrapper

`TabStripWrapper` adds a `TabStrip` to a base component.  This easily allows a
`TabStrip` to be applied to many kinds of components. The `Tabs` and
`LabeledTabs` use  `TabStripWrapper` to add a `TabStrip` to a `Modes` component,
but if visual or interactive effects more complex than `Modes` are desired, a
developer can create a component for that purpose and then add tabs to it with
`TabStripWrapper`.

This kind of wrapping will be common enough in Elix to warrant its own coding
pattern: a mixin that returns a new class incorporating a base class' template.

    class DivWithTabs extends TabStripWrapper(ShadowTemplateMixin(HTMLElement)) {
      get [symbols.template]() {
        return `
          <div id="container">
            <slot></slot>
          </div>
        `;
      }
    }
    customElements.define('div-with-tabs', DivWithTabs);

Here the base class' template just contains a single `div` holding a `slot`.
The `TabStripWrapper` will wrap this. The resulting `DivWithTabs` class will
have a template that looks like:

    <div-with-tabs>
      <TabStrip>
        <slot name="tabButtons"></slot>
      </TabStrip>
      <div id="container">
        <slot></slot>
      </div>
    </div-with-tabs>

The `TabStripWrapper` obtained the base class' template and injected it into
a template of its own that includes the `TabStrip` instance.

As shown in the complete (wrapped) template above, a component using
`TabStripWrapper` gains a slot called `tabButtons`. Child elements assigned to
that slot will be appear inside the `TabStrip`.

`TabStripWrapper` defines two properties, `tabAlign` and `tabPosition`, which
are wired directly to the corresponding on the inner `TabStrip` instance.
Additionally, the wrapper reflects the value of those properties as attributes:
`tab-align` and `tab-position`, respectively.


## LabeledTabButton

This is a simple button component intended to show a text label, and styled by
default to look like a classic tab.

The button supports a `tab-position` attribute that controls whether the tab
appears at the top, bottom, left or right edge of the panels it controls.
Visually, the tab button will have no interior border on the edge it shares with
the panels, so that the tab button and panel appear to exist on the same
surface. By default, the two corners opposite that edge are rounded in
skeumorphic reference to the tabs of real-world tabbed cardstock folders.


# Drawbacks

Why should we *not* do this? There are tradeoffs to choosing any path, please
attempt to identify them here. Consider the impact on the integration of this
feature with other existing and planned features, on the impact of the API churn
on existing apps, etc.


# Alternatives

What other designs have been considered? What is the impact of not doing this?


# Unresolved questions
