# Caret Navigation \- Explainer outlining need for specification

## Authors

- [Rakesh Goulikar](mailto:ragoulik@microsoft.com) (Microsoft)

## Participate

- Spec incubation issue: [w3c/editing\#529 — Explainer for Caret Movement Spec](https://github.com/w3c/editing/issues/529)  
- Web Platform Incubator Community Group (WICG): proposal issue (link forthcoming)  
- Web Editing Working Group: [w3c/editing issues labelled `caret-movement`](https://github.com/w3c/editing/issues?q=is%3Aissue+is%3Aopen+label%3Acaret-movement)

## Table of Contents

- [Introduction](#introduction)  
- [User-Facing Problem](#user-facing-problem)  
  - [Why this matters to everyone, not just editor developers](#why-this-matters-to-everyone-not-just-editor-developers)  
  - [Why this is hard to fix today](#why-this-is-hard-to-fix-today)  
  - [Goals](#goals)  
  - [Non-goals](#non-goals)  
- [The problem space in detail](#the-problem-space-in-detail)  
  - [1\. Bidirectional (BiDi / RTL) caret movement](#1-bidirectional-bidi--rtl-caret-movement)  
  - [2\. Addressability of empty elements and placeholders](#2-addressability-of-empty-elements-and-placeholders)  
  - [3\. Arrow-key navigation versus visual layout](#3-arrow-key-navigation-versus-visual-layout)  
  - [4\. Crossing editing-host and EditContext boundaries](#4-crossing-editing-host-and-editcontext-boundaries)  
  - [5\. Atomic inline non-editable content](#5-atomic-inline-non-editable-content)  
  - [6\. Word- and line-granularity movement and platform shortcuts](#6-word--and-line-granularity-movement-and-platform-shortcuts)  
- [Proposed Approach](#proposed-approach)  
- [Considered Alternatives](#considered-alternatives)  
- [Accessibility, Internationalization, Privacy, and Security Considerations](#accessibility-internationalization-privacy-and-security-considerations)  
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)  
- [References and acknowledgements](#references-and-acknowledgements)

## Introduction

The *caret* is the blinking vertical bar that shows where your next character will appear when you type. Moving that caret — with the arrow keys, by clicking, by jumping word-by-word, or by pressing Home and End — is one of the most basic things people do on the web. It happens billions of times a day in comment boxes, search fields, chat apps, email composers, and full word processors such as Google Docs, Microsoft Word for the web, and code editors such as VS Code for the web.

Despite being this fundamental, caret navigation is **not actually specified**. Each browser engine has grown its own behavior over two decades, and those behaviors quietly disagree with one another. The disagreements are usually small, but they are numerous, and they are most visible exactly where the web most needs to be reliable: in right-to-left and mixed-direction languages, in rich editors that build their own document models, and for users who rely on screen readers. This explainer proposes incubating a **Caret Navigation specification** that defines, in an interoperable and accessibility-aware way, *where the caret can be placed and how it moves* across editable and non-editable content. The goal is not to invent a new API for its own sake, but to write down and align the behavior that the platform already half-implements, so that authors and assistive technologies can finally depend on it.

## User-Facing Problem

### Why this matters to everyone, not just editor developers

Most people never think about the caret — and that is exactly the point. Caret movement should be predictable. When you press the Right arrow, the caret should go to the "next" place; when you press End, it should go to the end of the line; when you click in the middle of a word, the caret should land where you clicked. People build an unconscious muscle memory for these gestures and expect them to work the same everywhere.

That expectation breaks in several common situations:

- **For the more than half a billion people who read and write right-to-left scripts** — Arabic, Hebrew, Persian, Urdu, and others — the caret frequently moves in a way that feels backwards or jumps unexpectedly when a line mixes, say, Arabic with an English brand name or a number. The same document can behave differently in different browsers, so a sentence that is easy to edit in one browser becomes frustrating in another.  
    
- **For people who use screen readers**, the caret is not just visual: it is how the software decides what to announce next. When the caret's logical position and its visual position disagree (which happens constantly in mixed-direction text and in visually rearranged layouts), the spoken experience and the seen experience drift apart, and the user can no longer trust either one.  
    
- **For everyone using a modern web editor** — and that now includes mainstream productivity apps — the editor's developers have had to reverse-engineer and re-implement caret movement themselves, because the platform's built-in behavior is both inconsistent across browsers and not always exposed to them. Bugs in that re-implementation surface to ordinary users as carets that get "stuck," skip empty paragraphs, refuse to enter an empty heading, or land in the wrong spot after an image.

In other words, an under-specified caret produces real, user-visible papercuts, and they fall hardest on internationalization and accessibility — the two areas where the web most wants to be excellent.

### Why this is hard to fix today

A web editor cannot simply "ask the browser" to move the caret correctly and be done with it. Editors that maintain their own document model (ProseMirror, CKEditor, TipTap, Lexical, Svedit, the editors inside Google Docs and Microsoft Office for the web, and many others) routinely need to know things the platform does not tell them, such as:

- *If the user pressed "move one word forward" right now, where would the caret end up?* The answer depends on the operating system, the language, and the browser — and the editor only sees a raw key event, not the resolved intent.  
    
- *Is this empty paragraph a place the caret is allowed to rest?* Browsers answer differently, and sometimes answer differently for the **same** DOM depending on how that DOM was constructed.  
    
- *When the user presses the Right arrow at a direction boundary, or at the visual edge of an absolutely-positioned block, what is the next caret position?* There is no written-down rule, so each engine does something slightly different.

Because none of this is specified, editors ship large amounts of code to paper over the gaps, assistive technology vendors guess, and users get inconsistent behavior. The Web Editing Working Group has concluded that the durable fix is a dedicated specification, and — at the suggestion of the W3C team — to **incubate it in the WICG first** so that it attracts review and implementation interest from all three browser engines before it goes on the Recommendation track.

### Goals

This work aims to define **interoperable, accessibility-aware caret navigation** for editable content across both `contenteditable` and `EditContext`. Concretely, the specification should:

- Define a clear model for **how the caret moves** (by character, word, line, line boundary, and document boundary) and reconcile **logical (DOM) order with visual (on-screen) order**, including in **bidirectional text**.  
- Define **which positions the caret may occupy** — in particular, a precise, construction-independent rule for whether an **empty element** is a valid caret stop, and a consistent story for **placeholders**.  
- Define caret behavior across **structural boundaries**: between separate editing hosts and `EditContext` regions, and **around atomic inline non-editable content** such as images, SVG, and form controls (navigate before and after, but not into).  
- Define how the platform exposes **named navigation intents** (for example "move word backward," "move to start of line") to authors and assistive technology, rather than forcing editors to reverse-engineer per-OS key bindings.  
- Specify the **observable behavior** of all of the above so that engines, editors, and screen readers agree, while leaving room for legitimate language-specific tailoring.

A note on scope and ownership, the Working Group deliberately chartered this spec to cover *all* of the caret-movement scenarios it had previously identified and summarized — empty-element addressability, placeholders, visual-versus-logical arrow navigation, editing-host crossing, atomic inline content, and word/line granularity. The group's own short scope summary (drafted by Michael and Johannes Wilm) is reproduced in [The problem space in detail](#the-problem-space-in-detail) below, so this explainer tries to present the **full** problem space. It is understood that the detailed normative text for every scenario will be written collaboratively across the group over time, not by a single contributor.

### Non-goals

- **Re-specifying language-specific word segmentation from scratch.** Word boundaries are language-dependent; the spec should adopt [Unicode UAX\#29](https://unicode.org/reports/tr29/) as a sensible default and permit tailoring, not mandate a single segmentation algorithm for every script.  
- **Specifying the *visual rendering* of the caret** (its color, width, or how it paints over a placeholder). Those are real problems, but they belong with CSS UI and engine bug fixes; this spec depends on a stable rendering story and cross-references it rather than owning it.  
- **Replacing `contenteditable`, `EditContext`, Input Events, or the Selection API.** This spec sits alongside them and integrates with them; it does not supersede them.  
- **Mandating one specific keybinding per platform.** The spec defines navigation *intents* and their resulting positions, not which physical keys an operating system maps to those intents.

## The problem space in detail

When the Working Group agreed to start this specification, Michael and Johannes Wilm drafted a short summary of the scenarios it needs to cover. That summary is reproduced here (lightly edited for formatting) as the **agreed scope** of the work, so it is captured directly in the explainer rather than only paraphrased:

- **Bidirectional (bidi) text:** navigating bidirectional text sequences where visual order diverges from logical DOM order, ensuring arrow keys follow user-perceived directionality across mixed left-to-right and right-to-left scripts.  
- **Absolutely positioned editable regions:** traversing absolutely positioned editable elements that disrupt normal document flow, specifying how the caret should progress between visually adjacent regions irrespective of DOM hierarchy or CSS layout.  
- **Crossing editing hosts:** enabling seamless transitions of the caret between discrete editing hosts (for example, separate `contenteditable` containers or `EditContext` instances), with clear rules for focus movement across editable boundaries.  
- **Atomic inline non-editable content:** defining consistent behavior when moving around non-editable inline elements such as images, SVGs, or form controls, treating them as atomic units that the caret can navigate before and after, but not into, while preserving screen-reader compatibility and keyboard operability.  
- **Addressability of empty elements:** specifying which markup makes an "empty element" addressable with the caret (for example, `<p><br></p>` should be reachable while `<div></div>` is ignored).  
- **Placeholder labels:** standardizing the behavior and display of placeholder labels, which are inconsistent across browsers today.

The sections below expand each of these scenarios — plus the closely related question of **word- and line-granularity** movement that the group has also been discussing — into a user-facing symptom, a concrete example, a note on where browsers diverge, and the direction a specification should take. The intent is to make the case for *why* each area needs specifying; the full normative algorithms are future work for the group, to be written collaboratively.

### 1\. Bidirectional (BiDi / RTL) caret movement

**The symptom.** In text that mixes left-to-right and right-to-left scripts, the *logical* order of characters (the order they are stored in the document) and their *visual* order (the order they appear on screen) are not the same. The Unicode Bidirectional Algorithm rearranges runs of text for display. As a result, pressing the Right arrow can move the caret to a character that is visually to the *left*, selections can appear split into visually disconnected pieces, and the caret can "jump" across a boundary in a way that surprises the user. Different browsers resolve these moments differently, so the *same* mixed-direction sentence can be materially harder to edit in one browser than another.

**Why it is subtle.** At the seam between an Arabic phrase and an embedded English word or number, a single caret position on screen can correspond to two logical positions (this is often called caret *affinity*). What should happen when the user presses an arrow key at that seam is genuinely ambiguous unless it is written down, and today it is not.

**Direction for the spec.** Define caret movement at direction boundaries in terms of a clear model (logical, visual, or a defined hybrid), specify caret affinity at seams, and ensure arrow keys follow user-perceived directionality across mixed scripts. This work should draw on CSS Writing Modes and the Unicode Bidirectional Algorithm, and should be developed in coordination with the I18N Working Group. This is the area of most immediate interest to Microsoft, and a good candidate for the first concrete repros and tests.

### 2\. Addressability of empty elements and placeholders

**The symptom.** Editors constantly create empty blocks — an empty paragraph, an empty heading waiting for a title, an empty list item — and they want the caret to be able to rest there so the user can start typing. Whether an empty element is a valid caret stop, and whether you can *click* into it, varies by browser. Worse, in at least one engine the answer can depend on **how the DOM was built** rather than on the DOM itself: the identical structure can be reachable when written in HTML but unreachable when created programmatically.

**Concrete example.** The community has converged on a recommended markup for an empty, caret-addressable field:

\<div contenteditable="true"\>

  \<p\>\<br\>\</p\>

\</div\>

The `<br>` guarantees exactly one caret position and reserves line height while keeping the field visually empty. A placeholder can then be layered on without adding real text:

\<div contenteditable="true"\>

  \<p class="empty" placeholder="Enter a title"\>\<br\>\</p\>

\</div\>

\[contenteditable="true"\] \[placeholder\].empty::before {

  color: \#999;

  content: attr(placeholder);

}

The open question is the *rule*: a bare `<div></div>` arguably should **not** be a caret stop, while `<p><br></p>` should be — but engines disagree, and changing this risks breaking content that relies on today's behavior. There is also a distinct **hit-testing** problem: even where keyboard navigation works, clicking into an empty placeholder field can fail to place the caret or move focus.

**Direction for the spec.** Define caret-stop eligibility for empty elements as a **pure function of the resulting DOM and layout** (for example, tied to whether the element establishes a line box or has synthesized height), independent of construction history, and define **pointer-based** caret placement into empty fields, not just keyboard placement. Provide a standardized, in-flow placeholder pattern that does not break editing.

*Representative issues:* empty addressable fields and placeholders ([\#528](https://github.com/w3c/editing/issues/528), [\#503](https://github.com/w3c/editing/issues/503)); Firefox keyboard and click addressability ([\#500](https://github.com/w3c/editing/issues/500), [\#502](https://github.com/w3c/editing/issues/502)); construction-dependent caret stops ([\#513](https://github.com/w3c/editing/issues/513)); helper-node interference ([\#516](https://github.com/w3c/editing/issues/516)).

### 3\. Arrow-key navigation versus visual layout

**The symptom.** Modern editors use CSS — including absolute positioning — to place editable regions where they look right, which means the on-screen arrangement no longer matches DOM order. When the user presses an arrow key, should the caret follow the order of the underlying markup, or the order of what they see?

**Concrete example.** Consider an editable area with two absolutely-positioned blocks, one in the top-left and one in the top-right:

\<div contenteditable="true" style="position: relative; width: 500px; height: 500px;"\>

  \<div style="position: absolute; right: 0; top: 0;"\>text in top right corner\</div\>

  \<div style="position: absolute; left: 0; top: 0;"\>text in top left corner\</div\>

\</div\>

With the caret at the end of "…top left corner" and the user pressing the Right arrow, today the caret does not move (it is the last position in *DOM* order). A *visual* model would instead move it to just before "text in top right corner." Both answers are defensible; the platform simply has not chosen one. Vertical motion is even less consistent: Up/Down arrow navigation to and from absolutely-positioned blocks works in some engines and fails in others, even where Left/Right works.

**The catch.** A purely visual model has consequences elsewhere: selecting from the top-left block to the top-right block may require splitting the selection into multiple ranges to avoid a visually disjoint highlight, which ties this theme to the Selection API. And a visual model interacts directly with the bidirectional problem in Theme 1\.

**Direction for the spec.** Decide and specify whether caret motion is logical, visual, or a defined mode; define Up/Down line motion in the presence of out-of-flow positioned editable content; and reconcile the answer with both the BiDi model and multi-range selection.

*Representative issues:* visual-vs-logical movement ([\#533](https://github.com/w3c/editing/issues/533)); Up/Down across absolutely-positioned blocks ([\#523](https://github.com/w3c/editing/issues/523)).

### 4\. Crossing editing-host and EditContext boundaries

**The symptom.** A page can contain several independent editable regions — multiple `contenteditable` containers, or surfaces backed by `EditContext`. When the caret reaches the end of one region, what happens when the user keeps pressing the arrow key? Does it move into the next editable region, and does focus move with it? The non-editable gaps between regions make this even less defined.

**Direction for the spec.** Define how a single navigation gesture (for example, Right arrow at the end of editing host A) moves both the caret **and focus** into the adjacent editing host B, including how the caret traverses non-editable content in between. Because this behavior is intertwined with Input Events and `EditContext`, it should be specified in coordination with those specs rather than in isolation — a point the Working Group has emphasized.

### 5\. Atomic inline non-editable content

**The symptom.** Editable text often contains inline objects that are not themselves editable — an image, an `<svg>`, an emoji rendered as a widget, a `contenteditable="false"` mention chip, or a form control. Users expect the caret to move *to just before* and *just after* such an object, treating it as a single atomic unit, and never to get trapped *inside* it. Today the behavior of moving around these objects is inconsistent, and the accessibility mapping (what a screen reader announces as the caret passes the object) is not defined.

**Direction for the spec.** Define caret stops immediately before and after atomic inline non-editable content, prohibit caret stops inside it, and specify the corresponding accessibility behavior so screen-reader output stays correct and the object remains keyboard-operable.

### 6\. Word- and line-granularity movement and platform shortcuts

**The symptom.** "Move by word" (Option/Alt or Ctrl with an arrow) behaves differently across browsers *on the same operating system*, and differently across operating systems by design. For example, with the caret in `abc| def`, pressing Ctrl+Right lands at `|def` on Windows but `def|` on macOS — and on Windows, screen readers depend on the platform's word-breaking, so changing it has accessibility consequences. Separately, different platforms map *different keys* to the same intent (Home, Cmd+Left on macOS, and Alt+Left on Android can all mean "go to start of line"), yet a web editor only sees the raw key event with no indication of the resolved command, forcing it to maintain per-platform keybinding tables.

**Direction for the spec.** Two complementary moves. First, for *granularity*: specify the **observable** behavior of word and line movement — adopting UAX\#29 as a default while permitting language-specific tailoring — rather than mandating one algorithm, and respect operating-system and accessibility conventions. Second, for *intent exposure*: define a set of **named navigation intents** (move by character / word / line / line-boundary / document, in a given direction) and surface them to authors, most naturally through Input Events (`beforeinput`) and/or a selection-change hook, so editors can implement commands without reverse-engineering OS key bindings.

*Representative issues:* word-movement divergence ([w3c/editing\#278](https://github.com/w3c/editing/issues/278)); platform-specific shortcuts ([w3c/input-events\#180](https://github.com/w3c/input-events/issues/180)); selection-change hooks ([w3c/selection-api\#56](https://github.com/w3c/selection-api/issues/56), [\#37](https://github.com/w3c/selection-api/issues/37)).

## Proposed Approach

This explainer is at the **incubation** stage: its primary purpose is to frame the problem and attract multi-implementer review, not to lock down an API. The proposed approach therefore has two layers — a behavioral model that the platform should specify, and a small set of author-facing hooks — together with a list of open design questions the group must resolve.

**A behavioral model the platform specifies (mostly no new API).** Much of this work is about writing down behavior that already exists so engines converge. For authors and users, the "API" is simply that arrow keys, Home/End, and clicking *work the same everywhere*. The model needs to define, at minimum:

1. **A caret-stop eligibility rule** — given a DOM and its computed layout, the set of positions the caret may occupy, including the empty-element rule from Theme 2, computed as a pure function of state (so [\#513](https://github.com/w3c/editing/issues/513) cannot happen).  
2. **A movement model** — for each granularity (character, word, line, line boundary, document) and direction, the next caret position, including the logical-versus-visual decision (Themes 1 and 3\) and atomic-inline behavior (Theme 5).  
3. **Boundary-crossing rules** — caret and focus transfer between editing hosts and `EditContext` regions (Theme 4).

**Author-facing hooks (strawman, not yet decided).** Where editors *do* need to participate — because they maintain their own model — the platform should expose the user's resolved *intent* rather than raw keys. One shape under discussion is to dispatch named navigation commands through `beforeinput`, analogous to how editing commands are already surfaced:

// STRAWMAN — illustrative only; the exact event, names, and shape are open questions.

editorRoot.addEventListener("beforeinput", (e) \=\> {

  // The platform resolves the OS/locale key binding to a named intent

  // (e.g. "moveWordBackward", "moveToLineStart") and the position it would

  // produce, so the editor doesn't have to reverse-engineer key bindings.

  if (e.inputType \=== "moveWordBackward") {

    e.preventDefault();

    myEditorModel.moveCaret({ unit: "word", direction: "backward" });

  }

});

A complementary idea is a `beforeselectionchange`\-style event that reports the proposed next selection (with an intent hint) before it is applied, letting an editor answer the question *"if the user moved one word forward, where would the caret land?"* without re-implementing segmentation.

**Open design questions** the incubation must resolve (a non-exhaustive list):

- Is the canonical movement model **logical**, **visual**, or a defined hybrid/mode? (Drives Themes 1 and 3 and multi-range selection.)  
- The precise, construction-independent **caret-stop eligibility** rule for empty elements, and its relationship to line boxes and synthesized height.  
- **Hit-testing** into empty placeholders (pointer placement), not only keyboard.  
- Behavior of **Up/Down** motion across out-of-flow positioned editable content.  
- The exact mechanism and vocabulary for **named navigation intents**, and how they interact with OS conventions and screen-reader behavior.  
- How far to **standardize the algorithm versus the observable behavior** — the group's stated preference is to standardize observable behavior and use UAX\#29 as a default for word boundaries while allowing language tailoring.

**Integration points.** This work does not stand alone. It must be developed in coordination with **Input Events** (named intents via `beforeinput`), the **Selection API** (multi-range selection, `Selection.modify`, a possible `beforeselectionchange`), **`EditContext`** (caret/focus across decoupled editing surfaces), and **CSS** (caret rendering and, potentially, anchoring positioned UI to the caret/selection).

## Considered Alternatives

### Alternative 1: Do nothing — leave caret navigation unspecified

#### Pros

- No new specification work; engines keep their current behavior.

#### Cons

- The interoperability gaps, internationalization papercuts, and accessibility drift described above persist indefinitely.  
- Every editor keeps re-implementing and re-paying for the same workarounds.

#### Reason for rejection

This is the status quo, and it is precisely what is generating the long and growing list of `caret-movement` issues. The cost is paid repeatedly by authors, assistive technology, and end users.

### Alternative 2: Specify only bidirectional caret movement

#### Pros

- Smaller, faster, and aligned with Microsoft's most immediate interest.

#### Cons

- The Working Group explicitly chartered the spec to cover the broader set of caret-movement problems, and the themes are interdependent — for instance, the logical-versus-visual decision (Theme 3\) cannot be made sensibly without considering BiDi (Theme 1).  
- A BiDi-only document would leave the other well-documented interop failures unaddressed and would have to be reconciled later anyway.

#### Reason for rejection

Scoping to BiDi alone contradicts the group's decision and the technical reality that these problems share a single underlying model. BiDi remains the natural *starting point* for repros and tests, but not the boundary of the spec.

### Alternative 3: Leave it to JavaScript editor libraries

#### Pros

- No platform change; libraries already do this today.

#### Cons

- Libraries cannot fully observe platform intent (OS-resolved commands, word segmentation), so they guess and diverge.  
- Pushes accessibility-critical behavior into user code, where it is frequently implemented inconsistently or incompletely.

#### Reason for rejection

The need for editors to re-implement this is the *problem*, not the solution. Some behavior (caret-stop eligibility, OS-resolved intents) can only be provided correctly by the platform.

### Alternative 4: Defer entirely to the operating system

#### Pros

- Matches user muscle memory per platform and respects native conventions.

#### Cons

- The web must still define *observable* behavior so that engines on the same OS agree (they currently do not) and so that authors can reason about it.  
- Pure OS-deferral gives editors no portable, inspectable model and no way to query intent.

#### Reason for rejection

Respecting OS conventions is a goal, but it is not a substitute for a written-down, queryable, interoperable model. The spec can honor OS behavior while still specifying the observable result.

## Accessibility, Internationalization, Privacy, and Security Considerations

**Accessibility.** Accessibility is a primary motivation, not an afterthought. Screen readers use the caret to decide what to announce, and they depend on platform word-breaking on some operating systems (notably Windows). Any model must keep the caret's logical position, its visual position, and the assistive-technology announcement coherent — especially in bidirectional text and visually rearranged layouts — and must preserve keyboard operability around atomic inline content. Changes to word and line segmentation have direct accessibility consequences and must be evaluated with assistive technology in mind.

**Internationalization.** Bidirectional movement and word segmentation are inherently language-dependent. The spec should build on the Unicode Bidirectional Algorithm and adopt UAX\#29 as a default for word boundaries while explicitly allowing language-tailored improvements, so it does not freeze behavior in a way that harms scripts such as Japanese, Thai, Lao, or Khmer. This work should be coordinated with the I18N Working Group, which has already engaged on word-movement.

**Privacy and Security.** Caret navigation operates on content already present in the page and visible to the user, so it does not, by itself, expose new cross-origin or sensitive data. The main considerations are to ensure that any new author-facing hooks (such as named navigation intents or a selection-change event) do not leak information across origins — for example across iframe boundaries — and do not become a fingerprinting vector by revealing fine-grained OS or locale configuration beyond what is already observable. These should be assessed as the API shape is decided.

## Stakeholder Feedback / Opposition

- **Web Editing Working Group**: Positive. The group agreed to start this spec to cover both the bidirectional-text issues and the previously identified caret-movement issues, and chairs suggested incubating it in the WICG to broaden review ([w3c/editing\#529](https://github.com/w3c/editing/issues/529)).  
- **Browser engines**: Interest reported from all three engines represented at recent Editing calls; several behaviors have already been confirmed as engine bugs (with tracking bugs filed in Gecko and WebKit). Formal positions will be sought during incubation.  
- **I18N Working Group**: Engaged on word-segmentation and supportive of describing consistent behavior while preserving room for language-specific tailoring ([w3c/editing\#278](https://github.com/w3c/editing/issues/278)).  
- **Editor implementers**: The bulk of the documented issues come from real editor developers with live reproductions, indicating strong practitioner demand.

## References and acknowledgements

Many thanks for valuable feedback and guidance from (in alphabetical order):

- Johannes Wilm (Web Editing WG chair/editor)  
- Michael (Svedit)  
- Wenson Hsieh (Apple/WebKit)  
- Xiaoqian Wu (W3C)  
- and the participants of the W3C Web Editing Working Group

This proposal builds on prior art and ongoing discussions, including:

- [Selection API](https://w3c.github.io/selection-api/) and [Selection.direction with multiple ranges (\#171)](https://github.com/w3c/selection-api/issues/171)  
- [`contenteditable`](https://www.w3.org/TR/content-editable/)  
- [Unicode UAX\#29 — Text Segmentation](https://unicode.org/reports/tr29/)  
- [CSS Spatial Navigation explainer](https://drafts.csswg.org/css-nav-1/explainer)

Primary tracking and source issues:

- [w3c/editing\#529 — Explainer for Caret Movement Spec (anchor)](https://github.com/w3c/editing/issues/529)  
- [w3c/editing\#528 — Empty addressable content fields with placeholders](https://github.com/w3c/editing/issues/528)  
- [w3c/editing\#503 — Speccing placeholders (umbrella)](https://github.com/w3c/editing/issues/503)  
- [w3c/editing\#500](https://github.com/w3c/editing/issues/500) / [\#502](https://github.com/w3c/editing/issues/502) — Firefox empty placeholder addressability  
- [w3c/editing\#513 — Construction-dependent caret stops](https://github.com/w3c/editing/issues/513)  
- [w3c/editing\#516 — Empty span makes field unselectable (Safari)](https://github.com/w3c/editing/issues/516)  
- [w3c/editing\#533 — Caret movement by visual position vs DOM order](https://github.com/w3c/editing/issues/533)  
- [w3c/editing\#523 — Up/Down across absolutely-positioned blocks (Firefox)](https://github.com/w3c/editing/issues/523)  
- [w3c/editing\#278 — Moving the caret by one word](https://github.com/w3c/editing/issues/278)  
- [w3c/input-events\#180 — Platform-specific caret-navigation shortcuts](https://github.com/w3c/input-events/issues/180)
