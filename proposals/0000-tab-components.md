- Start Date: 2017-03-14
- RFC PR: (leave this empty)
- Elix Issues: (leave this empty)


# Summary

This proposes a set of components related to tabbed user interfaces. The two
main components can be readily used without modification or coding:

`Tabs` is a set of tabbed panels that can be navigated by selecting
corresponding tab buttons (which must be supplied by the developer). This
component takes care of the relative positioning of the tab buttons and the tab
panels. A typical example of `Tabs` being used for navigation:

    <elix-tabs>

      <button slot="tabButtons">Home</button>
      <div>Home page</div>

      <button slot="tabButtons">Search</button>
      <div>Search page</div>

      <button slot="tabButtons">Settings</button>
      <div>Settings page</div>

    </elix-tabs>

`LabeledTabs` is a specialized `Tabs` instance that presents simple tab buttons
with text labels. The text labels are drawn from the `aria-label` attribute of
the corresponding tab panels. Typical example:

    <elix-labeled-tabs>
      <div aria-label="General">General settings</div>
      <div aria-label="Accounts">Account settings</div>
      <div aria-label="Junk mail">Junk mail settings</div>
      <div aria-label="Signatures">Signature settings</div>
    </elix-labeled-tabs>

For flexibility, the components above are built up from secondary components:

* `TabStrip`. A row or column of tab buttons. Responsible for positioning the
  buttons, handling keyboard navigation, and supporting accessibility.

* `TabStripWrapper`. Adds a `TabStrip` to a base element, wiring the selection
  states of the two together. For example, the `Tabs` component uses
  `TabStripWrapper` to connect a `TabStrip` to a `Modes` instance.

* `LabeledTabButton`. A classic rounded tab button showing a text label for a
  tab panel. This is used internally by `LabeledTabs` for its tab buttons.

This RFC also includes the following mixins and helpers:

* `renderArrayAsElements`. Helper function to render one element for each item
  in an array.

* `ShadowReferencesMixin`. Exposes a `$` member on a component that references
  the elements in its shadow subtree, similar to Polymer's `$` feature.


# Motivation

Desired outcomes:

1. Developers can readily construct tabbed user interfaces that follow a
   conventional style.
2. Developers can easily adapt these components to created tabbed UIs for more
   specialized situations such as navigation UI.

Non-goals:

* Allow extensive customization of the tab components via properties. The tab
  pattern has many variations, and while it is tempting to address a large
  number of them with configurable properties, that generally leads to
  unmanageable complexity.


# Use cases

The `Tabs` component covers the common modal UI pattern in which the user can
directly control which modal state is presented. Tabs are typically used to
allow a UI to offer more controls than can fit in a confined area at a time.

* A common use case is Settings or configuration UIs. Here the classic look of
  a tabbed dialog or property sheet is addressed with `LabeledTabs`, although
  other looks are possible.
* Tabs may also be used in a main window to downplay less-commonly used aspects
  of a UI.
* Tabs are also an extremely navigation model. Many mobile applications present
  a navigation toolbar that behave like tabs, presenting 3–5 buttons that
  correspond to the app's main areas. In navigation use cases, the tab buttons
  typically have a toolbar button style rather than a classic tabbed appearance.

The `Tabs` and `LabeledTabs` components assume a standard tabbed UI design in
which clicking a tab immediately makes the corresponding tab panel visible. To
manage that visual transition, those components rely internally on an instance
of `Modes`. However, there are use cases in which an app wishes to provide other
visual effects for the transition between panels, such as a sliding animation.
To accommodate those use cases, the `Tabs` components are constructed from
lower-lever parts that can be easily recombined to add tab panel functionality
to a component displaying a more complex visual transition.


# Detailed design

These components are designed to comply as closely as possible with the
accessibility recommendations for [WAI-ARIA Authoring Practices for
Tabs](https://www.w3.org/TR/wai-aria-practices-1.1/#tabpanel).


## Tabs

The `Tabs` component implements a standard tab panel UI pattern. It is nothing
more than a `TabStrip` (below) wrapped around a `Modes` instance: selecting a
tab button in the tab strip tells the Modes component to display the
corresponding panel. The actual work of wrapping a `TabStrip` around another
element is performed by `TabStripWrapper`.

`Tabs` exposes two slots: 1) a `tabButtons` slot for the buttons that will be
shown in the tab strip, and 2) a default slot for the tab panels. The sample in
the "Summary" (above) illustrates the use of these slots. That sample will
render three tab panels below a tab strip with three corresponding buttons. The
buttons for the `tabButtons` slot could also be grouped together; all that
matters is that they appear in the same order as the corresponding tab panels.

See `TabStripWrapper` for a description of the properties exposed by `Tabs`.


## LabeledTabs

A `LabeledTabs` can be used in the common case where the tab buttons present a
simple text label, such as in a Settings UI.

`LabeledTabs` is simply a subclass of `Tabs` that fills the default content of
`tabButtons` slot with a collection of `LabeledTabButton` instances (below). It
creates one `LabeledTabButton` for each panel, and sets the `textContent` of the
button to the `aria-label` attribute of the corresponding panel.

`LabeledTabs` assigns a default ARIA role of `tab` to the tab buttons.


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

`TabStrip` assigns a default ARIA role of `tab` to each button.


## TabStripWrapper

`TabStripWrapper` adds a `TabStrip` to a base component. This allows a
`TabStrip` to be applied to many kinds of components. The `Tabs` and
`LabeledTabs` use  `TabStripWrapper` to add a `TabStrip` to a `Modes` component,
but if visual or interactive effects more complex than `Modes` are desired, a
developer can create a component for that purpose and then add tabs to it with
`TabStripWrapper`.

This kind of wrapping will be common enough in Elix to warrant its own wrapper
pattern. A wrapper is a mixin that returns a new class incorporating a base
class' template:

    class DivWithTabs extends TabStripWrapper(ShadowTemplateMixin(HTMLElement)) {
      get [symbols.template]() {
        return `
          <!-- Defined by DivWithTabs -->
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

      <!-- Defined by DivWithTabs -->
      <div id="container">
        <slot></slot>
      </div>

    </div-with-tabs>

The `TabStripWrapper` obtained the `DivWithTabs` template and wrapped it with
content of its own that includes a `TabStrip` instance.

As shown in the complete (wrapped) template above, a component using
`TabStripWrapper` gains a slot called `tabButtons`. Child elements assigned to
that slot will be appear inside the `TabStrip`.

`TabStripWrapper` defines two properties, `tabAlign` and `tabPosition`, which
are wired directly to the corresponding on the inner `TabStrip` instance.
Additionally, the wrapper reflects the value of those properties as `tab-align`
and `tab-position` attributes, respectively.

`TabStripWrapper` handles the assignment of ARIA roles necessary to support best
practices. It assigns a default ARIA role of `tablist` to the component itself
and `tabpanel` to each tab panel.


## LabeledTabButton

This is a simple button component intended to show a text label, and styled by
default to look like a classic tab.

The button supports a `tab-position` attribute that controls whether the tab
should style itself appropriately for appearing at the top, bottom, left or
right edge of the panels it controls. Visually, the tab button will have no
interior border on the edge it shares with the panels, so that the tab button
and panel appear to exist on the same surface. By default, the two corners
opposite that edge are rounded in skeumorphic reference to the tabs of
real-world tabbed cardstock folders.
