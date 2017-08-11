- Start Date: 2017-07-12
- RFC PR: https://github.com/elix/rfcs/pull/11
- Elix code PR: (leave this empty)

# Summary

This proposal recommends adding a family of mixins and components to Elix for handling overlay patterns, including popups, dialogs, menus, tooltips, dropdowns, and drawer panels. The result includes components which are ready to use in common scenarios, as well as constituent mixins, base classes, and other helpers so developers can address more challenging scenarios.

# Motivation

Overlay patterns are extremely common in user interfaces, but the web platform provides no good primitives for creating them. The result is that developers often struggle to implement such patterns. Providing mixins and components for these patterns is a high priority for Elix.

Desired outcomes:

* Developers can quickly create popups, dialogs, and other components that reliably appear over the rest of the currently-visible elements.
* Our overlay strategy works with the browser as much as possible, and avoids reimplementing browser-level features when possible.
* Developers can use Elix overlay mixins and helpers to create their own solutions.
* Elix overlay components provided a good user experience by default for all users, including keyboard users and users of screen readers.

Non-goals:

* Provide the One Dialog Component to Rule Them All, or the One Menu, etc. There’s too much variability in how applications want to treat these UI elements to provide components that work for everybody. We will focus on common scenarios.

* Guarantee that overlays appear on top of all visible elements in 100% of cases. The browser simply doesn’t provide good ways of programmatically inspecting the page to determine what element is currently topmost and ensuring that a new element is above that element. Developers facing complex edge cases may still gain some advantage by using Elix helpers and mixins to address their special circumstances.

# Use cases

Common types of overlays include:

* Popup. A lightweight overlay that displays information, often positioned relative to some other visible element. These typically remain visible until dismissed, although the user can readily dismiss them by clicking outside of them. Other user actions such as scrolling, resizing the window, or switching to a different window implicitly close the popup.
* Dropdown. A popup that usually appears below a read-only element, presenting a set of options to the user. Selecting an option typically replaces the current value in the element.
* Menu button. A dropdown attached to a button.
* ContextMenu. A dropdown attached to an element, invoked on right click or long press / 3D Touch.
* Menu bar. A collection of menus. This is more complex than a row of menu buttons — the user can use the keyboard to navigate between the menus. The user can make a selection from the menu in a single mouse operation (mouse down, drag to desired item, then mouse up) or in two (mouse click, move to desired item, mouse click). The menus also support riffing behavior: if the user opens one menu, they can move to another to immediately open it.
* ComboBox. A dropdown attached to an interactive input element. Selecting an option from the combo box puts that option’s text into the input element. The user can also enter choices of their own. As the user types, the text auto-completes against the set of options in the dropdown. The input element is accessible while the overlay is visible so that the user can edit its text. On mobile, a combo box renders as an input panel (docked to the bottom of the viewport, like a touch keyboard); the component ensures the input element is not covered by the panel.
* Drawer. See the Material Design guidelines for [Navigation Drawer](https://material.io/guidelines/patterns/navigation-drawer.html) for sample specs.
* ToolTip. A popup that appears adjacent to an element when the user hovers the mouse over that element; the ToolTip disappears when the mouse moves away. Supports riffing behavior: moving the mouse from one ToolTip-enabled element to a second such element immediately displays the second element’s ToolTip.
* Toast. A temporary alert communicating non-critical information, such as that a background event has occurred or an operation has completed. By default, it slides up from the bottom of the viewport, but other placements are possible.
* Dialog. A heavyweight modal overlay. These are typically positioned with respect to the viewport (e.g., centered). The user must explicitly dismiss the dialog in order to close it. Scrolling, resizing the window, or switching to a different window does not close the dialog.
* Alert dialog. A dialog which presents a message or question to the user and offers a small set of response options such as Yes/No or OK/Cancel.

Not all these use cases are addressed by the initial implementation of this proposal.


# General factoring

Elix’s implementation strategy for overlays is the same as for all other types of components: we decompose the problem into a set of mixins, wrappers, and other helpers that can be combined to produce a variety of results. This proposal breaks the general problem of overlays into the following mixins and wrappers:

* OpenCloseMixin. Adds an `opened` property and `open`/`close` methods.
* OverlayMixin. Ensures that, when an element is opened, it is in the document and generally on top of all other existing elements.
* DialogModalityMixin. Imposes dialog-style modality on an overlay. (See Dialog Modality, below.) Requires a backdrop.
* PopupModalityMixin. Imposes popup-style modality on an overlay. If the user clicks on the page background or manipulates the window (blur/resize/scroll), the overlay implicitly closes.
* TransitionEffectMixin. Invokes an asynchronous transition when the overlay is opened/closed, following a standard sequence of before/during/after timings.
* BackdropWrapper. Adds a backdrop element to a template for overlays like dialogs that want to block access to the page UI while the overlay is visible.
* FocusCaptureWrapper. For any component that wants to capture the keyboard focus. (See the section on "Capturing keyboard focus", below.)

Elix will also provide ready-to-use elements which are stock combinations of the above. The initial set is:

* Dialog. Element combining the mixins for standard modal dialogs.
* AlertDialog. A Dialog with a small set of response buttons.
* Drawer. A container, typically docked to the side of the screen, providing navigation and other commands at small screen sizes.
* Popup. Element combining the mixins for lightweight overlays.
* Toast. A popup that generally appears at the bottom of the screen, and closes itself after a specified time interval.

The rest of this proposal examines each mixin/element in detail.


# Example

To illustrate how the overlay mixins cooperate, let’s follow the sequence of operations that occur when an Elix `Drawer` is opened. The `Drawer` class includes a number of mixins, but for this example assume it’s defined with just four mixins:

    class Drawer extends 
      DialogModalityMixin(
      OpenCloseMixin(
      OverlayMixin(
      TransitionEffectMixin(
        HTMLElement
      )))) { ... }

Given this definition, when the `opened` property on a `Drawer` is set to true, the following sequence of operations will be executed in order (top to bottom) by the mixins and the component:

<table>
  <tr>
    <td>Drawer class</td>
    <td>DialogModality
Mixin</td>
    <td>OpenCloseMixin</td>
    <td>OverlayMixin</td>
    <td>TransitionEffect
Mixin</td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td>`opened` setter:
Stores new value of `opened`.
Invokes `openedChanged`.
Invokes async `showEffect`.</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>`showEffect`:
Invokes `beforeEffect`.</td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td></td>
    <td>`beforeEffect`:
Saves currently focused element.
Sets `z-index`.
Makes overlay visible.
Gives overlay focus.</td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td>`beforeEffect`:
Hides page scroll bars.</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>`showEffect`:
Invokes `applyEffect`.
`applyEffect`:
Start listening for transitioneend event.
Applies CSS classes "effect" and “opening”.</td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td>`opened` setter:
Raises opened-changed event.
[Synchronous work is now done.]</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>Component styling triggers the sliding “opening” transition.
Async transition occurs. When finished, raises the
transitionend event.</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>`transitionend` listener:
Resolves promise of `applyEffect`.

`showEffect` resumes:
Invokes `afterEffect`.</td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>`afterEffect`:
Removes CSS classes “effect” and “opening”.</td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td>`afterEffect`:
Adds CSS class “opened”.
Resolves `whenOpened` promise.
[Async work is now done.]</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>Component styling for “opened” class keeps drawer open.</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>

In this example, each mixin is handling a different aspect of opening the drawer. `OpenCloseMixin` handles the basic semantics of something becoming "opened". `OverlayMixin` ensures the drawer is visible and on top of things. `TransitionEffectMixin` coordinates the timing of the work required before, during, and after the drawer’s sliding transition. And `DialogModalityMixin` temporarily hides the page scroll bars while the drawer is open. The `Drawer` component itself doesn’t provide any code here; it simply provides appropriate CSS styling.


# OpenCloseMixin

This mixin provides a consistent public API for components that open and close, including overlays and various types of expandable/collapsable elements.

The mixin provides the following members:

* `opened` property that is true when open, false when closed.
* `open`/`close` methods that set the `opened` property and return a promise for when the open/close action has completed (including any async effects).
* `toggle` method which toggles the opened property.
* `whenOpened`/`whenClosed` promises for the next time the element opens/closes.

If the component defines the following optional members, the mixin will take advantage of them:

* Effect methods compatible with TransitionEffectMixin if the element wants to define async opening/closing effects. The use of transition effects is not required. If a component doesn’t use `TransitionEffectMixin` or a compatible mixin, then `OpenCloseMixin` will perform its work synchronously, with no transition effects.

* `symbols.openedChanged` method that will be invoked when the opened property changes.

The `OpenCloseMixin` is designed to support user interface elements that have two states that can be described as "opened" and “closed”. These can be grouped into two top-level categories:

1. Elements that open over other elements — that is, overlays. This is the first category of element that will use `OpenCloseMixin` (below).
2. Elements that expand and collapse inline — these may be panels that open to reveal more detail, or list items that expand to show more detail or additional commands. These elements will be designed and implemented later.


## `opened` property

This property is true if the element is open and false when closed. The default value is false.

Changing the value of `opened` has the following effects, in this order:

* If `opened` is being set to true, a new promise is created that will resolve the next time the element is closed. Conversely, if `opened` is being set to false, a new promise is created that will resolve when the next time the element is opened.
* If the element defines a method called `symbols.openedChanged`, that method is invoked, passing in the new Boolean value of `opened`. This allows other mixins or the component class to respond to changes in `opened`.
* If a component supports asynchronous effects (typically by using `TransitionEffectMixin`), the method `symbols.showEffect` is invoked. If `opened` is changing to true, the requested effect will be the "opening" effect; if `opened` is changing to false, the requested effect will be “closing”.
* Special case of the above: if you open an element by default through an `opened="true”` attribute in markup, `TransitionEffectMixin` will _not_ render any asynchronous opening effect. It assumes you want to immediately render the element as opened.
* If `opened` is being set to true, the "opened" CSS class is added to the element. If `opened` is being set to false, the “opened” CSS class is removed from the element.
* If the value of `opened` is changing in response to internal component activity, the "opened-changed" event is raised.

Setting the `opened` property is a synchronous operation, but changing the property’s value can trigger asynchronous operations such as opening and closing effects. To wait for the opening or closing operation to complete, use `whenOpened` or `whenClosed`, respectively.


## `open` and `close` methods

The `open` and `close` methods are alternative ways of setting the `opened` property to true or false, respectively, but they also offer additional features.

The `open` method returns a promise that resolves when the element has finished opening, including the completion of any asynchronous effects. Likewise, the `close` method returns a promise that resolves when the element has finished closing.

The `close` method also accepts a `result` parameter that indicates the result of closing the element. For example, a modal dialog component can use this parameter to indicate a choice the user made inside the dialog.


## `toggle` method

The toggle method simply toggles the value of the `opened` property from true to false or vice-versa.


## `whenOpened` and `whenClosed` methods

The `whenOpened` and `whenClosed` methods return promises for the next time the element is opened or closed, respectively. This may be more convenient than listening for the "opened-changed" event, especially if you are only interested in listening for the next occurrence of that event with a particular `opened` value.

For example, the code to process the result of a notification dialog might use the `whenClosed` method:

      const dialog = document.createElement('elix-notification-dialog');
      dialog.textContent = 'Hello, world';
      dialog.choices = ['Yes', 'No', 'Cancel'];
      dialog.open();
      dialog.whenClosed().then(result => {
        console.log(`You picked ${result}.`);
      });


# OverlayMixin

The purpose of OverlayMixin is to make an opened element appear on top of other page elements, then hide or remove it when closed. This mixin and `OpenCloseMixin` form the core overlay behavior for Elix components.

The mixin expects the component to provide:

* An invocation of `symbols.beforeEffect` and `symbols.afterEffect` methods for both "opening" and “closing” effects. This is generally implemented by using `OpenCloseMixin`.

The mixin provides these features to the component:

1. Appends the element to the DOM when opened, if it’s not already in the DOM.
2. Can calculate and apply a default z-index that in many cases will be sufficient to have the overlay appear on top of other page elements.
3. Makes the element visible before any opening effects begin, and hides it after any closing effects complete.
4. Remembers which element had the focus before the overlay was opened, and tries to restore the focus there when the overlay is closed.
5. A `teleportToBodyOnOpen` property that optionally moves an element already in the DOM to the end of the document body when opened. This is intended only to address challenging overlay edge cases; see the discussion below.

If the component defines the following optional members, the mixin will take advantage of them:

* An effect API compatible with `TransitionEffectMixin`. This allows an element to provide opening/closing effects. The effects are _not_ applied if the `opened` property is set in markup when the document is loading. The use of transition effects is not required. It is not necessary for a component to use `TransitionEffectMixin` or a compatible mixin. In that case, `OverlayMixin` will simply perform its work synchronously.

All other aspects of overlay behavior are handled by other mixins and wrappers. For example, addition of a backdrop element behind the overlay is handled by `BackdropWrapper`. Intercepting and responding to UI events is handled by `PopupModalityMixin` and `DialogModalityMixin`. Management of asynchronous visual opening/closing effects are provided by `TransitionEffectMixin`.


## Background

The job of `OverlayMixin` is to make an overlay appear on top of all other currently-visible elements. Unfortunately, that is a difficult problem. This section summarizes some of the key issues.


### z-index and stacking contexts

In HTML, the _z-order_ determines which elements visually overlap others and therefore appear to be closer to the user. Z-order is determined by a variety of factors: relative document order, assigned z-index values, and stacking contexts. A stacking context is a layer or group of elements that move forward or backward together. The document body is one stacking context; others are implicitly created by certain CSS attributes.

It’s reasonable for you, as a web developer, to prefer to avoid such arcane topics. It would be nice to have an overlay component that can simply guarantee to you that its contents will be topmost. But if you are going to create web apps where elements overlap, or create components for web apps, you are better off mastering these topics. See [What no one told you about z-index](https://philipwalton.com/articles/what-no-one-told-you-about-z-index/) or [The stacking context](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Positioning/Understanding_z_index/The_stacking_context) for primers.

Shadow DOM introduces a new level of complexity: a host element may not have a z-index, but an element within the host’s shadow subtree may have a z-index. That means that elements hidden inside a shadow can affect the z-order. It’s not only expensive to examine shadow elements for z-index — the existence of closed shadow roots makes that impossible.

So even before setting out to tackle putting overlays on top of other visible elements, it should be understood that _there is no perfect solution_. The best Elix can do is come up with an overlay mechanism that works predictably and reasonably well.


### Complex edge cases

Here are some tricky overlay situations that have come up in practice:

1. An overlay is sitting inside of a container that establishes a stacking context, but is behind other elements. An [example](http://jsbin.com/tocoxax/edit?html,output) from Valdrin Koshi at Google attempts to create an overlay that appears over an element that’s already on top. The overlay fails because it sits inside a container that forms a new stacking context. The container has no z-index, and it doesn’t matter how high the overlay’s z-index is — the container is in the background, which means the overlay inside it will be too. The general [solution](http://jsbin.com/nukuku/edit?html,output) is to ensure all containing elements that establish a stacking context get a `position`, and a `z-index` at least as high as the other element with a z-index.

2. An overlay is inside an element with overflow: hidden. See this [example](https://jsfiddle.net/Saulis/efktmgt1/) from Sauli Tähkäpää at Vaadin. The core issue here is not that the overlay isn’t on top, but that it is being clipped by the use of overflow: hidden. One [solution](https://jsfiddle.net/Jan_Miksovsky/2ot9xsh7/) is to position the overlay with position: fixed, breaking the overlay out of the clipped region. Another solution for overlays in large scrolling regions is to simply ensure the overlay doesn’t try to extend beyond the region’s bounds. In a scrollable region, for example, a dropdown menu button near the bottom of the scrolling area might be able to drop _up_ instead of down.

3. An overlay is inside a container that has overflow: hidden _and_ a transform other than `none`. This is a more complex version of the above problem. Here the use of position: fixed on the overlay will not help, because the container’s transform causes it, rather than the viewport, to clip the overlay. (See the [MDN documentation for position: fixed]([https://developer.mozilla.org/en-US/docs/Web/CSS/position](https://developer.mozilla.org/en-US/docs/Web/CSS/position)).)


### Overlays always in the DOM vs temporarily in the DOM

Stepping back, it seems there are two general categories of overlays:

1. **An overlay that is always in the DOM**, even when closed. The overlay may sit inside the shadow tree of a component alongside a visible element that triggers the appearance of the overlay. The overlay’s on-top aspect might be a side effect of an "opened" property being true. This category of overlay might be broken into two subcategories:

    a. An overlay attached to a component. Example: a combo box whose shadow tree contains an overlay which the combo box itself opens and closes. As long as the combo box element is on the page, the overlay is on the page too.
    b. An overlay that sits in the light DOM for convenience because it’s not otherwise associated with anything. Example: a confirmation dialog sitting hidden on a page that’s invoked to confirm some operation on the page. Strictly speaking, the dialog isn’t part of the core page UI, but the developer puts in on the page because it’s a convenient place to stash markup which the page might need.

2. **An overlay temporarily added to the DOM when it’s shown**, then removed when its closed. Such overlays might be invoked with an open() method rather than by setting an opened property to true. An example might be a OK/Cancel notification dialog.

The stacking context question above isn’t as acute for the second category (overlays temporarily added to the DOM). The overlay must already be added to the DOM, so it may as well be added to the end of the document body, where it can avoid some of the complications of stacking contexts. Additionally, if the developer is invoking some open() method to open the overlay, it’s not too surprising if the overlay adds itself to the end of the document.

The requirement to understand stacking context is more pressing for the first category, in which an overlay is always sitting inside the DOM.


### Style inheritance

A related (but slightly less pressing question) is ensuring that an overlay picks up appropriate styling. As with z-order, this question should be considered in light of the two categories above:

1. An overlay that is always in the DOM. These inherit styling according to their DOM placement. Sometimes that is desirable, sometime it isn’t. For an example of the latter, consider a menu button component that produces a popup menu overlay. Someone adding this menu button to a toolbar might apply styling to ensure the button blends in with the other buttons on the toolbar. But the developer may not want the button’s popup menu to pick up that background color.
2. An overlay temporarily added to the DOM. If these are added to the top level of the document, they’ll pick up only the page’s top-level styles. This is likely consistent with the developer’s expectation.
As with the z-ordering topic, the first category presents more challenges. That said, it’s not entirely clear whether the presence of overlays in the discussion changes much. The present state of web component theming is quite weak, but the answers that do exist — CSS Custom Properties and @apply mixins — apply equally well to the specific case of overlays as they do to web components in general.


## Opening overlays

As a result of the issues raised above, `OverlayMixin` takes the following approach:

1. For overlays that are always in the DOM, opening an overlay doesn’t normally change its DOM location. Generally speaking, we adhere to the principle of least astonishment. Since stacking contexts are complex, we will document the conditions under which an overlay can appear on top of other elements. We can provide guidance for applying CSS to containing elements to ensure the overlay can appear on top.
2. Opening an overlay that is _not_ already in the DOM (the second category discussed earlier) causes it to be added to the end of the document body. The overlay is removed when closed.
3. To address challenging edge cases, `OverlayMixin` supports teleportation via an opt-in property, `teleportToBodyOnOpen`. If the overlay is already in the DOM when opened, and `teleportToBodyOnOpen` is true, the mixin moves the overlay to end of the document body. When the overlay is closed, the mixin returns the overlay to its original position in the DOM.

The mixin actually opens an overlay when an "opening" effect is applied. See `TransitionEffectMixin` for details. When `symbols.beforeEffect(“opening”)` is invoked, `OverlayMixin` sets things up and ensures the overlay is ready for the effect to be applied:

1. Remembers which element had the focus before the overlay was opened.
2. If the overlay isn’t already in the DOM, the overlay is added to the DOM.
3. If the overlay is already in the DOM, and `teleportToBodyOnOpen` is true (see below), the overlay is temporarily moved to the end of the document body.
4. If the overlay does not already have a z-index applied to it, a default z-index is applied. This will be 1 greater than the largest z-index currently applied to any element already in the document’s light DOM. In very large documents where this calculation may be costly, you can simply apply a z-index to the overlay at design time to circumvent this calculation.
5. Makes the overlay visible. Note that this is generally necessary for any CSS transitions or web animations to be visible.
6. Gives the overlay focus.

To see the complete sequence of work that occurs when an overlay opens, including work performed by other mixins, refer to the Example in the overview at the top of this document.


## Closing overlays

Conversely, `OverlayMixin` closes an overlay when a "closing" effect is applied. The mixin does work both before and after the effect. Before the “closing” effect is applied with `symbols.beforeEffect(“closing”)`, this mixin does the following:

1. Restores focus to the element that had focus before the overlay was opened.

After the "closing" effect has been applied and `symbols.afterEffect(“closing”)` is invoked, `OverlayMixin` cleans up:

1. Hide the overlay. Note that this is done _after_ the effect is applied so that the overlay will remain visible while the asynchronous closing effect is doing its work. (Otherwise the user would always see the overlay disappear instantly.)
2. If a default z-index was applied to the overlay during opening, the default z-index is removed.
3. If the overlay was added to the DOM during opening, the overlay is removed from the DOM.
4. If the overlay was teleported to the document body during opening, the overlay is returned to its original location in the DOM.


## `teleportToBodyOnOpen` property: moving overlays in the DOM

By default, `OverlayMixin` assumes that: 1) applying a default z-index is usually sufficient to get an overlay to appear on top of other elements, and 2) overlays in more complex situations can be made to appear on top of other elements by correctly applying CSS properties to manage stacking contexts. However, as noted earlier, certain hard edge cases make it difficult to ensure that an overlay appears on top of other visible elements.

Demo: https://janmiksovsky.github.io/elix/demos/dialogTrapped.html

One brute-force means to address those problems is to dynamically move or "teleport" an opening overlay in the DOM from its original location to the end of the document body. In that location, it is more likely (but not guaranteed) to appear on top of other elements. An invisible placeholder element is left in the DOM to mark the overlay’s original location. When the overlay is closing, the overlay is teleported back to that original location, and the placeholder is removed.

This strategy can work but has disadvantages:

* The silent movement of the overlay within the document is at least as surprising as the way stacking contexts work.
* The overlay can receive different styles when it is moved to a new location.
* The overlay being moved may contain a `<slot>` provided by a host. When the overlay is moved, the slotted content will be dropped. See http://jsbin.com/honoze/edit?html,output.

This complexity on top of complexity works against the grain of the browser, and is avoided by default. However, since teleporting overlays may be useful in certain cases, `OverlayMixin` supports an opt-in ability to teleport overlays.

The mixin defines a property called `teleportToBodyOnOpen` which, when explicitly set to true, will move an opening overlay already in the DOM to the end of the document body, then move the overlay back to its original location when closing.

Demo: https://janmiksovsky.github.io/elix/demos/dialogTeleport.html

The `teleportToBodyOnOpen` property is false by default to avoid surprising developers. Additionally, the property is reflected to an attribute called `teleport-to-body-on-open` as a self-explanatory hint (e.g., to other people who come across an overlay in the DOM with that property set) as to why the overlay magically moves when opened.


# DialogModalityMixin

This mixin blocks various user interactions to make an overlay behave like a modal dialog. This mixin is generally used in conjunction with a backdrop (such as that provided by `BackdropWrapper`).

Expects the component to provide:

* An open/close API compatible with OpenCloseMixin.

The mixin provides these features to the component:

* Disables scrolling on the background page.
* A default ARIA role of `dialog`.
* Closes the element if user presses the Esc key.

For modeless overlays, see `PopupModalityMixin` instead.


## Degrees of modality

Overlays often impose a degree of modality: user interactions whose results have different results than usual (i.e., a mode) while the overlay is open. An interaction may be blocked, or have the result of dismissing the overlay. Different degrees of modality are appropriate for different types of overlays.

In the case of Elix, a given degree of modality can be achieved by applying a corresponding mixin, although the developer is free to supply their own modality.

<table>
  <tr>
    <td>Interaction</td>
    <td>Dialog modality</td>
    <td>Popup modality</td>
    <td>Menu modality</td>
  </tr>
  <tr>
    <td>Click on background page</td>
    <td>Blocked (1)</td>
    <td>Closes</td>
    <td>Closes (click is absorbed)(2)</td>
  </tr>
  <tr>
    <td>Move focus to background page</td>
    <td>Blocked</td>
    <td>Closes</td>
    <td>Closes</td>
  </tr>
  <tr>
    <td>Window blur</td>
    <td>(No effect)</td>
    <td>Closes</td>
    <td>Closes</td>
  </tr>
  <tr>
    <td>Window resize</td>
    <td>(No effect)</td>
    <td>Closes</td>
    <td>Closes</td>
  </tr>
  <tr>
    <td>Window scroll</td>
    <td>Blocked</td>
    <td>Closes</td>
    <td>Closes</td>
  </tr>
  <tr>
    <td>Esc key</td>
    <td>Closes</td>
    <td>Closes</td>
    <td>Closes</td>
  </tr>
</table>

Notes:

(1) `DialogModalityMixin` does not block background clicks on its own. For that, include a backdrop in your component’s shadow root. See `BackdropWrapper`.

(2) Menus generally supply a variant of popup modality: background page clicks close the menu, but are absorbed so that they don’t reach any controls that happen to be under the mouse/touchpoint. Elix will address the need for such a modality when it tackles menus.


## Avoiding background scrolling

We’d like overlays with `DialogModalityMixin` to prevent the user from manipulating the page in the background. This includes preventing scrolling of the page while the dialog is open. This goal is complicated by the fact that a scroll event is _not_ cancelable.

`DialogModalityMixin` prevents scrolling by temporarily hiding scroll bars on the document body. This is done by temporarily applying `overflow="hidden”` to the document body while the overlay is open. When the dialog is closed, the mixin restores the previous overflow value. Removing the body’s scroll bars definitively removes the possibility of the user scrolling the page via touch, trackpad, and keyboard while the overlay is open, while still allowing the user to scroll within the overlay.

Hiding the body’s vertical scroll bar can have the unfortunate side effect of changing the body width, which in turn can cause distracting and undesirable text reflow. This is primarily an issue for Edge and IE, where scroll bars consume screen real estate by default. The other browsers use scroll bars which float above the page, and do not normally consume space. (But if the dev explicitly styles page scroll bars using pseudo-selectors, then the scroll bars _will_ consume space.)

To avoid content reflow, `DialogModalityMixin` calculates the width of the body scroll bar and temporarily increases document’s right padding to match that width. (If the scroll bar floats above the page and does not consume any horizontal space, this step is skipped.) As a result, the user should not see content reflow then the overlay opens or closes.

Note: As an implementation alternative to disabling scroll bars, we investigated the possibility of having the overlay listen for touchmove and mousewheel events that might cause scrolling and prevent those events from doing their default work, as well as absorbing keyboard events that might cause the page to scroll (Space, Page Up/Down, Home/End, Up/Down, and Left/Right). Unfortunately, disabling UI events with scroll potential has the side effect of also disabling scrolling for any elements within the dialog itself. Example: A modal dialog contains a scrollable textarea. Pressing Page Down when that textarea has focus should scroll it. That means the keyboard event should _not_ be absorbed when it arises from within the dialog. However, when the user has scrolled the textarea to the bottom, the textarea will begin ignoring the Page Down key and let the document handle it — causing the page to scroll. For this reason, we deemed it more reliable to disable document scroll bars while an overlay is open.


# PopupModalityMixin

This mixin makes an overlay behave like a popup by dismissing it when certain user interactions occur.

Expects the component to provide:

* An open/close API compatible with OpenCloseMixin.

The mixin provides these features to the component:

* Event handlers that close the element if the user clicks outside the element, presses the Esc key, moves the focus outside the element, scrolls the document, resizes the document, or switches focus away from the document.
* A default ARIA role of `alert`.

For modal overlays, use `DialogModalityMixin` instead. See the documentation of that mixin for a comparison of modality behaviors.


## Wrapping focus

Current [WAI-ARIA practices for modal dialogs](https://www.w3.org/TR/wai-aria-practices/#dialog_modal) recommend wrapping focus within the overlay. This recommendation may reflect what’s currently feasible in a web app rather than what keyboard users actually want. Trapping focus within a dialog means the user has no way to tab to the browser’s own chrome controls, including the address bar, which seems really bad.

For reference, there have been proposals to add an [`inert` attribute](https://github.com/alice/inert) and/or a [`blockingElements` primitive](http://robdodson.me/building-better-accessibility-primitives). Those would be very helpful, but even if they happen someday, that won’t be for a long time.

Wrapping keyboard focus is generally assumed to mean that, if the user presses Tab on the last focusable element in an overlay, focus moves to the first focusable element. Conversely, if the user presses Shift+Tab on the first focusable element, the focus moves to the last focusable element. However, implementing such wrapping is actually quite challenging, and ultimately requires attempting to reproduce the browser’s Tab and Shift+Tab behaviors in JavaScript. That is complex and error-prone.

To avoid such complexity, `FocusCaptureWrapper` relaxes the requirements slightly by adding the overlay itself into the tab cycle:

* When an overlay is opened, focus is given to the overlay itself, rather than the first focusable elements with the overlay. For example, opening a dialog will put the focus on the dialog. Aside from being easy to implement, it has the benefit that a screen reader will announce that the appearance of the dialog and read aloud its ARIA label (typically the dialog title).
* When the overlay has focus, the user can press Tab to move the focus to the first focusable element. The browser handles the change in focus, so there’s no work for `FocusCaptureWrapper` to do. If the overlay has focus and the user presses Shift+Tab, focus moves to the last focusable element. As it turns out, implementing that behavior is not particularly difficult.
* When the user reaches the last focusable element, pressing Tab wraps the focus to the overlay again (instead of the first focusable element).

Note that, while `FocusCaptureWrapper` is designed for use in modal overlays, it’s not directly linked to the Elix overlay mixins such as `OverlayMixin`. You may also find use for it in other situations.


# TransitionEffectMixin

This mixin enables asynchronous visual effects by applying CSS classes that can trigger CSS transitions. It provides a standard timing model so that work can be performed both before and after the asynchronous effects run.

Expects the component to provide:

* Styling with CSS transitions triggered by the application of CSS classes.

The mixin provides these features to the component:

* A `symbols.showEffect` method that invokes the following methods on the component in order: `symbols.beforeEffect`, `symbols.applyEffect`, and `symbols.afterEffect`.
* A `symbols.applyEffect` method implementation that applies CSS classes to the component host to trigger the start of the CSS transition.
* Suppresses effect application if requested before an element’s connectedCallback is invoked.
* Suppresses effects if the user has expressed an accessibility preference for reduced motion. See https://webkit.org/blog/7551/responsive-design-for-motion/.

If the component defines the following optional members, the mixin will take advantage of them:

* `symbols.beforeEffect` method which runs synchronously before applyEffect is invoked.
* `symbols.afterEffect` method which runs synchronously after applyEffect has completed.
* `symbols.elementsWithTransitions` method that returns an array of elements that will be affected by the transitions.

The timing model imposed by `TransitionEffectMixin` is designed to be replicated in other mixins. For example, Elix expects to eventually create a mixin to trigger effects that use the Web Animations API. That mixin will use the same timing model so that it could be used as a drop-in replacement for `TransitionEffectMixin`.

To see how `TransitionEffectMixin` can work in the context of an overlay element with CSS transition effects, refer to the Example in the overview at the top of this document.


## Using strings to name effects

`TransitionEffectMixin` identifies visual effects with string names. You can pick whatever effect name are meaningful. Since the effect name will be applied to the component host to trigger the corresponding CSS transition, effect names should be valid CSS class names.

To reflect the fact that the effect is a transformation, Elix suggests naming your effects as dynamic actions ending in "-ing": “opening”, “closing”, “sending”, “receiving”, etc.


## `symbols.showEffect(effect)` method

This method is used to tell `TransitionEffectMixin` to exhibit the named effect. The mixin will invoke the following methods in order:

1. `symbols.beforeEffect`
2. `symbols.applyEffect`. This will return a Promise; the mixin will wait for this promise to resolve before proceeding.
3. `symbols.afterEffect`.

There are cases where the mixin will _skip_ the invocation of `applyEffect`:

* If the user has expressed an accessibility preference for reduced motion.
* If the component’s `connectedCallback` has not yet been invoked. That can happen if the component has not yet been attached to the DOM, or if the component was defined in page markup.

In these cases, the mixin _will_ invoke `beforeEffect` and `afterEffect`, but will skip `applyEffect`. This allows a component to leave itself in the same final state it would have if the asynchronous effect had happened. The `Drawer` component takes advantage of that to allow an author to initially open a Drawer in markup through the `opened="true”` attribute. The Drawer will be shown already open from the time the page loads, with no distracting initial transition.


## `symbols.beforeEffect(effect)` method

If a component defines this method, the mixin invokes it before triggering the CSS transition with `symbols.applyEffect` (below) The `beforeEffect` method takes a single parameter: the string name of the effect about to be applied. This allows the component or other mixins to prepare the component to undergo the effect.

`OverlayMixin`, for example, uses `symbols.beforeEffect` to prepare for the "opening" effect by ensuring that the component is visible.


## `symbols.applyEffect(effect)` method

The mixin invokes this method to apply two CSS classes to the component that trigger the desired CSS transitions:

1. The first CSS class is the string "effect".
2. The second CSS class is the name of the effect being applied ("opening", “closing”, etc.).

This method returns a Promise for the completion of the CSS transitions.


## `symbols.afterEffect(effect)` method

The mixin provides an implementation of this method that removes the CSS class names added by `applyEffect` (above).

Your component or mixin can do additional work at this point to finalize the application of an effect. A common thing to do is to apply a CSS class that keeps the element in the end state of the transition.

Note that, if you would like to leave the element in the final state of the CSS transition, you will need to define style rules that 


## `symbols.elementsWithTransitions(effect)` method

Use this optional method to let `TransitionEffectMixin` know which elements are affected by a given CSS transition.

To wait for effects to finish, `TransitionEffectMixin` listens for `transitionend` events. These events will be raised by the component itself or any element in its shadow DOM subtree. Depending upon the needs of the effect, a transition may affect more than one element, in which case multiple `transitionend` events will fire. The mixin will listen for the first `transitionend` and, when that is fired, consider the effect to have completed.

For this reason, you will generally want an effect to have the same duration on all affected elements, so that they all finish at the same time. Once the mixin hears the first `transitionend` event, it will ignore subsequent events, and immediately invoke `symbols.afterEffect`.

In native Shadow DOM, it’s possible for the mixin to listen to all `transitionend` events generated by the component by attaching a listener to the component host. However, in polyfilled ShadyDOM, `transitionend` events do not bubble as expected. Therefore, if a component has transitions that affect any elements within the component’s shadow tree, the mixin must be told which elements are affected by the transition.

To do this, define a `symbols.elementsWithTransitions` method. This takes a parameter indicating the effect being invoked. Have the method return an array of the element(s) exhibiting CSS transitions as part of that effect.


# BackdropWrapper

This mixin wraps a component’s template to add a backdrop element suitable for use in modal overlays such as dialogs. The backdrop plays two roles: the it prevents background clicks, and it can be styled (with, for example, a semi-transparent background color) to help focus the user’s attention away from the page background and toward the overlay. This wrapper is often used in conjunction with `DialogModalityMixin`.

Expects the component to provide:

* A template-stamping mechanism compatible with ShadowTemplateMixin.

The wrapper provides these features to the component:

* A container element identified as `#overlayContent` that holds the element’s primary content.
* A backdrop element identified as `#backdrop` that sits behind the primary content and covers the viewport.

By default, `BackdropWrapper` provides no styling of the backdrop. If you are using this wrapper in your component, the elements added by the wrapper will be in your component’s shadow tree, so you can style them like any shadow element. For example, to add a classic semi-transparent black to the backdrop and a white background and shadow to the overlay content, you could do the following:


    const Base =
      BackdropWrapper(
      DialogModalityMixin(
      OverlayMixin(
      OpenCloseMixin(
      ShadowTemplateMixin(
        HTMLElement
      )))));

    class MyDialog extends Base {
      [symbols.template]() {
        return super[symbols.template](`
          <style>
            #backdrop {
              background: black;
              opacity: 0.2;
            }
            #overlayContent {
              background: white;
              border: 1px solid rgba(0, 0, 0, 0.2);
              box-shadow: 0 2px 10px rgba(0, 0, 0, 0.5);
            }
          </style>
          <slot></slot>
        `);
      }
    }


Note the invocation of `super[symbols.template]`. That passes the template string to the `BackdropWrapper` so it can be wrapped with the desired elements.


# FocusCaptureWrapper

This mixin wraps a component’s template such that, once the component gains the keyboard focus, Tab and Shift+Tab operations will cycle the focus within the component.

Expects the component to provide:

* A template-stamping mechanism compatible with ShadowTemplateMixin.

The mixin provides these features to the component:

* Template elements and event handlers that will cause the keyboard focus to wrap.


# Dialog

This component presents its children as a basic modal dialog which appears on top of the main page content and which the user must interact with before they can return to the page.

Dialog uses `BackdropWrapper` to add a backdrop behind the main overlay content. Both the backdrop and the dialog itself can be styled. A simple example:

Demo: https://janmiksovsky.github.io/elix/demos/dialog.html

Your dialogs should include a prominent button or other element the user can use to close the dialog and return to the page. By default, `DialogModalityMixin` also provides the Esc key as a keyboard shortcut to close the dialog. Within the dialog, `FocusCaptureWrapper` is used to keep the keyboard focus within the dialog while it is open.

Dialogs often provide a result back to the page. In such cases, you can arrange for your dialog to return a result by passing a parameter to the `close` method.

By default, no visual effects are provided for the opening/closing of the dialog. You can add these yourself by applying `TransitionEffectMixin` to `Dialog`.

Sometimes all a modal dialog needs to do is get the user’s answer to a question with a small number of possible responses. For such dialogs, consider using `AlertDialog`.


## Usage

Although modal dialogs are common in user interfaces, exercise caution when introducing them. They tend to interrupt the user’s workflow and can cause frustration. See "Disadvantages of Modal Dialogs" in this discussion of [Modal and Nonmodal Dialogs]([https://www.nngroup.com/articles/modal-nonmodal-dialog/]). That page offers the following guidelines:

1. Use modal dialogs for important warnings, as a way to prevent or correct critical errors.
2. Use modal dialogs to request the user to enter information critical to continuing the current process.
3. Modal dialogs can be used to fragment a complex workflow into simpler steps.
4. Use modal dialogs for ask for information that, when provided, could significantly lessen users’ work or effort.
5. Do not use modal dialogs for nonessential information that is not related to the current user flow.
6. Avoid modal dialogs that interrupt high-stake processes such as checkout flows.
7. Avoid modal dialogs for complex decision making that requires additional sources of information unavailable in the modal.


# AlertDialog

An `AlertDialog` is a form of `Dialog` designed to ask the user a single question and let them respond by clicking one of a small number of buttons labeled with text.

Demo: https://janmiksovsky.github.io/elix/demos/alertDialog.html

For keyboard support, the user can select a choice by pressing the initial character of its text label. Example: typing "o" or “O” will press a button labeled “OK”.

`AlertDialog` is a subclass of `Dialog`, and so inherits its API and features.


## `choices` property

An array of strings. For each string in the array, the `AlertDialog` displays a button labeled with that string.

You can use any strings for the choices. `AlertDialog` provides static properties offering two simple arrays of choices for common situations:

* `OK`: an array with the single choice "OK".
* `OK_CANCEL`: an array with two choices, "OK" and “Cancel”.

You can use these to set the `choices` property, or you can provide custom choices:

    // Create an OK/Cancel alert.
    const alert = new AlertDialog();
    alert.choices = AlertDialog.OK_CANCEL;


# Drawer

A drawer is a modal container generally used to provide navigation in situations where: a) screen real estate is constrained and b) the navigation UI is not critical to completing the user’s primary goal (and, hence, not critical to the application’s business goal).

Demo: https://janmiksovsky.github.io/elix/demos/drawer.html

A drawer is often used as part of a responsive design, in which the links in a toolbar or menu that is always shown at desktop window sizes collapse to a slide-out navigation drawer shown on mobile:

Demo: https://janmiksovsky.github.io/elix/demos/responsiveDrawer.html

A user click/tap on the drawer’s backdrop implicitly dismisses the drawer. (Compare with `Dialog`, where backdrop clicks are ignored.)

Elix plans to eventually add touch gestures to Drawer so that users can close it (and possibly open it) with a swipe.


# Popup

A `Popup` is a lightweight form of overlay that, when opened, displays its children on top of other page elements.

Demo: https://janmiksovsky.github.io/elix/demos/popup.html

By default, a `Popup` is centered in the viewport. For other types of positioning (e.g., to position a popup relative to another element on the page), you will need to provide styling.

A `Popup` incorporates `PopupModalityMixin`, so most UI interactions (clicking, scrolling, etc.) outside the popup will dismiss it. The user can explicitly dismiss the popup by pressing the Esc key.

The `Toast` element is similar to `Popup` and is specifically designed for showing a short, non-critical message.


# Toast

A lightweight popup intended to display a short, non-critical message until a specified `duration` elapses or the user dismisses it.

Demo: https://janmiksovsky.github.io/elix/demos/toast.html

For a general popup element, see `Popup`.


## `duration` property

This property specifies in milliseconds how long a toast should remain open before being implicitly closed. The default value is 2500 milliseconds (2.5 seconds).

To support interactivity within a toast, the timer is disabled if the user moves the mouse inside the toast or taps within it. When/if the user later moves the mouse outside the toast, or taps outside it, the timer will be restarted at zero.

Setting `duration` to 0 or `null` will disable the timer, allowing the toast to remain open indefinitely.


## `fromEdge` property

The `fromEdge` property determines the edge from which the toast will slide into view. Supported values are:

* "bottom" (the default): slides in from the center of the bottom of the window.
* "bottom-left"
* "bottom-right"
* "top"
* "top-left"
* "top-right"

The `Toast` component supports right-to-left languages such as Arabic and Hebrew. If the effective value of the element’s `dir` attribute is set to "rtl" (right to left), then the interpretation of the `fromEdge` property will be flipped horizontally: for example, setting `from-edge=“top-right”` will cause the `Toast` to appear from the top _left_.
