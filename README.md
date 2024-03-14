# Explainer for the Document-Isolation-Policy API

**Instructions for the explainer author: Search for "todo" in this repository and update all the
instances as appropriate. For the instances in `index.bs`, update the repository name, but you can
leave the rest until you start the specification. Then delete the TODOs and this block of text.**

This proposal is an early design sketch by the Chrome Web Platform Security team to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

TODO: Fill in the whole explainer template below using https://tag.w3.org/explainers/ as a
reference. Look for [brackets].

## Authors

- [Camille Lamy](https://github.com/camillelamy)

## Participate
- https://github.com/explainers-by-googlers/document-isolation-policy/issues

## Table of Contents [if the explainer is longer than one printed page]

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [User research](#user-research)
- [Use cases](#use-cases)
  - [Use case 1](#use-case-1)
  - [Use case 2](#use-case-2)
- [[Potential Solution]](#potential-solution)
  - [How this solution would solve the use cases](#how-this-solution-would-solve-the-use-cases)
    - [Use case 1](#use-case-1-1)
    - [Use case 2](#use-case-2-1)
- [Detailed design discussion](#detailed-design-discussion)
  - [[Tricky design choice #1]](#tricky-design-choice-1)
  - [[Tricky design choice 2]](#tricky-design-choice-2)
- [Considered alternatives](#considered-alternatives)
  - [[Alternative 1]](#alternative-1)
  - [[Alternative 2]](#alternative-2)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Developers want to build applications that are fast using [SharedArrayBuffers](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer) (SAB), which can improve computation time by ~40%. But SharedArrayBuffers allow to create high-precision timers that can be exploited in a [Spectre](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)) attack, allowing to leak cross-origin user data. To mitigate the risk, SharedArrayBuffers are gated behind [crossOriginIsolation](https://developer.mozilla.org/en-US/docs/Web/API/crossOriginIsolated) (COI). CrossOriginIsolation requires to deploy both [CrossOriginOpenerPolicy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Opener-Policy) (COOP) and [CrossOriginEmbedderPolicy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Embedder-Policy) (COEP). Both have proven hard to deploy, COOP because it prevents communication with cross-origin popups, and COEP because it imposes restrictions on third-party embeds. Finally, the whole COOP + COEP model is focused on providing access to SharedArrayBuffers to the top-level frame. Cross-origin embeds can only use SABs if their embedder deploys crossOriginIsolation and delegates the permission to use COI-gated APIs, making the availability of SABs in third-party iframes very unreliable.

The API proposed in this document, Document-Isolation-Policy, is proposing to solve these deployment concerns by relying on the browser [Out-of-Process-Iframe](https://www.chromium.org/developers/design-documents/oop-iframes/) capability. It will provide a way to securely build fast applications using SharedArrayBuffers while maintaining communication with cross-origin popups (needed for OAuth and payment flows) and not requiring extra work to embed cross-origin iframes. Finally, it will be available for embedded widgets as well as top-level frames, allowing to build efficient compute heavy widgets that are embedded across a variety of websites (e.g. photo library, video conference iframe, etc…).


## Goals

We want to provide end users with web applications that can perform compute heavy tasks efficiently (e.g. video games, videoconferencing, photo editing) and make use of the composability of the web (to allow for more convenient OAuth/payment flows, to embed cross-origin widgets such as a social media sharing widget,...). And we want this to happen without introducing security flaws in the web platform that would allow malicious applications to gain access to cross-origin user data.

## Non-goals

While the original [crossOriginIsolation](https://developer.mozilla.org/en-US/docs/Web/API/crossOriginIsolated) proposal was agnostic to the platform [SiteIsolation](https://www.chromium.org/Home/chromium-security/site-isolation/) capability, this proposal will rely on a browser's ability to isolate iframes in a different process from their embedder.

## Use cases

### Use case 1:

The user would like some of their compute heavy web apps to be faster (e.g. games, spreadsheet, photo editing, videoconferencing). To make these apps faster, relying on multithreading and shared memory is needed, but this means having access to SharedArrayBuffers. Currently, a web app cannot use shared memory and cross-origin popups at the same time due to the restriction imposed by crossOriginIsolation. This means the web app cannot use 3rd party OAuth or payment flows.

### Use case 2

The user would like some of their compute heavy web apps to be faster (e.g. games, spreadsheet, photo editing, videoconferencing). To make these apps faster, relying on multithreading and shared memory is needed, but this means having access to SharedArrayBuffers. Currently, a web app using shared memory is limited in the 3rd party iframes that it can embed. For example, it cannot embed personalized ads unless the ads do a lot of work to support COEP themselves. Personalized ads are the revenue model for a lot of free web apps, so that means those free web apps cannot use shared memory unless the whole ad ecosystem deploys COEP.

### Use case 3

The user would like some of the embedded widgets on the page they browse to be faster (e.g. photo library widget, map widget, video chat widget, ...). To make these widgets faster, relying on multithreading and shared memory is needed, but this means having access to SharedArrayBuffers. But access to SharedArrayBuffers is only possible if the embedder of the widget deploys crossOriginIsolation (with the constraints described in the previous use cases). So a widget embedded in a wide variety of websites cannot use shared memory to improve its performance.

## [Potential Solution]

[For each related element of the proposed solution - be it an additional JS method, a new object, a new element, a new concept etc., create a section which briefly describes it.]

```js
// Provide example code - not IDL - demonstrating the design of the feature.

// If this API can be used on its own to address a user need,
// link it back to one of the scenarios in the goals section.

// If you need to show how to get the feature set up
// (initialized, or using permissions, etc.), include that too.
```

[Where necessary, provide links to longer explanations of the relevant pre-existing concepts and API.
If there is no suitable external documentation, you might like to provide supplementary information as an appendix in this document, and provide an internal link where appropriate.]

[If this is already specced, link to the relevant section of the spec.]

[If spec work is in progress, link to the PR or draft of the spec.]

[If you have more potential solutions in mind, add ## Potential Solution 2, 3, etc. sections.]

### How this solution would solve the use cases

[If there are a suite of interacting APIs, show how they work together to solve the use cases described.]

#### Use case 1

[Description of the end-user scenario]

```js
// Sample code demonstrating how to use these APIs to address that scenario.
```

#### Use case 2

[etc.]

## Detailed design discussion

### [Tricky design choice #1]

[Talk through the tradeoffs in coming to the specific design point you want to make.]

```js
// Illustrated with example code.
```

[This may be an open question,
in which case you should link to any active discussion threads.]

### [Tricky design choice 2]

[etc.]

## Considered alternatives

[This should include as many alternatives as you can,
from high level architectural decisions down to alternative naming choices.]

### [Alternative 1]

[Describe an alternative which was considered,
and why you decided against it.]

### [Alternative 2]

[etc.]

## Stakeholder Feedback / Opposition

[Implementors and other stakeholders may already have publicly stated positions on this work. If you can, list them here with links to evidence as appropriate.]

- [Implementor A] : Positive
- [Stakeholder B] : No signals
- [Implementor C] : Negative

[If appropriate, explain the reasons given by other implementors for their concerns.]

## References & acknowledgements

[Your design will change and be informed by many people; acknowledge them in an ongoing way! It helps build community and, as we only get by through the contributions of many, is only fair.]

[Unless you have a specific reason not to, these should be in alphabetical order.]

Many thanks for valuable feedback and advice from:

- [Person 1]
- [Person 2]
- [etc.]
