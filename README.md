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

Developers want to build applications that are fast using [SharedArrayBuffers](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer) (SAB), which can improve computation time by ~40%. But SharedArrayBuffers allow to create high-precision timers that can be exploited in a [Spectre](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)) attack, allowing to leak cross-origin user data. To mitigate the risk, SharedArrayBuffers are gated behind [crossOriginIsolation](https://developer.mozilla.org/en-US/docs/Web/API/crossOriginIsolated) (COI). CrossOriginIsolation requires to deploy both [Cross-Origin-Opener-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Opener-Policy) (COOP) and [Cross-Origin-Embedder-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Embedder-Policy) (COEP). Both have proven hard to deploy, COOP because it prevents communication with cross-origin popups, and COEP because it imposes restrictions on third-party embeds. Finally, the whole COOP + COEP model is focused on providing access to SharedArrayBuffers to the top-level frame. Cross-origin embeds can only use SABs if their embedder deploys crossOriginIsolation and delegates the permission to use COI-gated APIs, making the availability of SABs in third-party iframes very unreliable.

The API proposed in this document, Document-Isolation-Policy, is proposing to solve these deployment concerns by relying on the browser [Out-of-Process-Iframe](https://www.chromium.org/developers/design-documents/oop-iframes/) capability. It will provide a way to securely build fast applications using SharedArrayBuffers while maintaining communication with cross-origin popups (needed for OAuth and payment flows) and not requiring extra work to embed cross-origin iframes. Finally, it will be available for embedded widgets as well as top-level frames, allowing to build efficient compute heavy widgets that are embedded across a variety of websites (e.g. photo library, video conference iframe, etc…).


## Goals

We want to provide end users with web applications that can perform compute heavy tasks efficiently (e.g. video games, videoconferencing, photo editing) and make use of the composability of the web (to allow for more convenient OAuth/payment flows, to embed cross-origin widgets such as a social media sharing widget,...). And we want this to happen without introducing security flaws in the web platform that would allow malicious applications to gain access to cross-origin user data.

## Non-goals

While the original [crossOriginIsolation](https://developer.mozilla.org/en-US/docs/Web/API/crossOriginIsolated) proposal was agnostic to the platform [SiteIsolation](https://www.chromium.org/Home/chromium-security/site-isolation/) capability, this proposal will rely on a browser's ability to isolate iframes in a different process from their embedder.

## Use cases

### App with cross-origin popup

The user would like some of their compute heavy web apps to be faster (e.g. games, spreadsheet, photo editing, videoconferencing). To make these apps faster, relying on multithreading and shared memory is needed, but this means having access to SharedArrayBuffers. Currently, a web app cannot use shared memory and cross-origin popups at the same time due to the restriction imposed by crossOriginIsolation. This means the web app cannot use 3rd party OAuth or payment flows.

### App with 3rd party iframe

The user would like some of their compute heavy web apps to be faster (e.g. games, spreadsheet, photo editing, videoconferencing). To make these apps faster, relying on multithreading and shared memory is needed, but this means having access to SharedArrayBuffers. Currently, a web app using shared memory is limited in the 3rd party iframes that it can embed. For example, it cannot embed personalized ads unless the ads do a lot of work to support COEP themselves. Personalized ads are the revenue model for a lot of free web apps, so that means those free web apps cannot use shared memory unless the whole ad ecosystem deploys COEP.

### Embedded widget

The user would like some of the embedded widgets on the page they browse to be faster (e.g. photo library widget, map widget, video chat widget, ...). To make these widgets faster, relying on multithreading and shared memory is needed, but this means having access to SharedArrayBuffers. But access to SharedArrayBuffers is only possible if the embedder of the widget deploys crossOriginIsolation (with the constraints described in the previous use cases). So a widget embedded in a wide variety of websites cannot use shared memory to improve its performance.

## Background: Process wide XS-Leaks and the crossOriginIsolated model

### Same-origin policy

The same-origin policy is a critical security mechanism in the web that helps to isolate different websites and prevent them from accessing each other's data. It works by restricting the interaction between web pages based on their origin, which is determined by the combination of the scheme (e.g., HTTP or HTTPS), the host name, and the port number.

Under the same-origin policy, web pages can only access data from other pages that have the same origin. This helps protect user data and prevents malicious websites from accessing sensitive information, such as passwords or credit card numbers.

There are a few exceptions to the same-origin policy, such as cross-origin resource sharing (CORS). However, these exceptions are carefully designed to preserve the security of user data.

The same-origin policy is an essential part of web security, and it helps to protect users from a variety of attacks. By preventing websites from accessing each other's data, the same-origin policy helps to keep user data safe and secure, and allows the web to remain composable.

### Process wide XS-Leaks
Unfortunately, several web APIs allow an attacker to bypass the same-origin policy and to leak information from authenticated cross-origin resources. These kinds of vulnerabilities are known as XS-Leaks. In particular, we will focus on process-wide XS-Leaks, which can affect any resource loaded in the same renderer process as an attacker.

Process-wide XS-Leaks can be direct. For example, [performance.measureUserAgentSpecificMemory](https://developer.mozilla.org/en-US/docs/Web/API/Performance/measureUserAgentSpecificMemory) allows to easily leak a resource size. Leaking a resource size can be exploited in various cross-site search attacks that target resources that vary based on query parameters.

Process-wide XS-Leaks can also take the form of a [Spectre](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)) attack. Spectre is a security vulnerability affecting modern processors, enabling attackers to read arbitrary data from the process they are executing in. It exploits speculative execution, where processors execute instructions before receiving all necessary data, to access and steal sensitive information. This includes authenticated cross-origin resources loaded in the process. Spectre attacks’ efficiency is correlated to timer resolution. More precise timers make Spectre attacks faster. This includes direct timers like [performance.now](https://developer.mozilla.org/en-US/docs/Web/API/Performance/now), but also timers constructed through SharedArrayBuffers.

In the absence of [Site Isolation](https://www.chromium.org/Home/chromium-security/site-isolation/), an attacker can load a wide variety of authenticated cross-origin resources in their process.

*Without Site Isolation, all resources (documents + subresources) for a Browsing Context Group are rendered in the same process.*

![Iframes, subresources and popups are placed in the same process without SiteIsolation](/images/background1.png)

All of the cross-origin resources loaded into a process are vulnerable, unless they load through [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)[^1]. This is because cross-origin CORS requests are sent without credentials by default. In order to make credentialled requests, the cross-origin endpoint has to opt into the request being credentialled and the content shared with the cross-origin requestor.

[^1]: Note that the CORS caveat only applies to subresources directly loaded by the attacker origin. All other subresources are still at risk.

*Without Site Isolation, all non-CORS subresources and all documents are at risk from an attacker.*

![Without SiteIsolation, all resources are at risk from an attacker](/images/background2.png)

### The cross-origin isolation model
The [crossOriginIsolation](https://developer.mozilla.org/en-US/docs/Web/API/crossOriginIsolated) model was developed as a way to address the process-wide XS-Leak threat without requiring Site Isolation and CORS for all subresources.

First, cross-origin isolation identifies APIs that pose higher risk of process wide XS-Leaks and gate them behind a set of policies (COOP and COEP). These APIs are:
- [performance.measureUserAgentSpecificMemory()](https://developer.mozilla.org/en-US/docs/Web/API/Performance/measureUserAgentSpecificMemory)
- high resolution [performance.now()](https://developer.mozilla.org/en-US/docs/Web/API/Performance/now)
- [SharedArrayBuffers](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)

[Cross-Origin-Embedder-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Embedder-Policy) (COEP) addresses the threat to subresources and embedded iframes. It requires non-CORS subresources to either opt into being loaded in the COEP context (COEP require-corp). Or it loads them without credentials (COEP credentialless).

*COEP protects subresources by asking for an opt-in or loading cross-origin resources without credentials.*

![COEP asks subresources for an opt-in or load them without credentials](/images/background3.png)

In the absence of Out-of-process-iframes, iframes are loaded in the same-process as their parent. To protect cross-origin iframes from their embedder, COEP requires cross-origin iframes embedded by a COEP document to opt into being embedded by sending a CORP header.

For the model to work, COEP needs to apply to all documents in the process. So COEP requires documents embedded in a COEP document to also deploy COEP.

Because this is complex to deploy, Chrome also introduced credentialless iframes. A credentialless iframe allows a COEP document to embed documents without COEP. However, those documents only have access to an ephemeral storage partition, initially blank. This ensures that they do not have access to stored credentials. This is safe in the XS-Leak threat model because we are concerned with personalized user data.

*COEP protects cross-origin resources of the page.*

![COEP ensures all resources of a page are protected](/images/background4.png)

[Cross-Origin-Opener-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Opener-Policy) (COOP) protects other pages that would normally be loaded in the same process as an attacker without Site Isolation. COOP same-origin triggers a browsing context group switch when navigating to a page with a different top-level origin and/or COOP. Pages in a different browsing context group cannot communicate with each other, so it is safe to place them in different processes, even without Out-Of-Process-IFrames (OOPIF).

The proposed [COOP restrict-properties](https://github.com/WICG/coop-restrict-properties) restrict WindowProxy properties available to documents in pages with a different top-level origin. It also prevents synchronous DOM access between documents in pages with a different top-level origin. The latter part allows the browser to place the two pages in different processes.

*COOP protects other pages from an attacker by restricting access to pages opened.*

![COOP restricts access to popups](/images/background5.png)

When the top-level frame enforces both COOP and COEP, the page becomes cross-origin isolated. It gains access to powerful but leaky APIs.

*CrossOriginIsolation provides a secure model to gain access to leaky APIs even in the absence of OOPIF.*

![COI is a safe model for powerful APIs](/images/background6.png)

Finally, the last piece of the puzzle is a [permission policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Permissions_Policy). By default, cross-origin iframes do not have access to COI-gated APIs, even if their parent has enabled COI. The parent must delegate the permission for them to gain access to the APIs. This ensures that the parent is adequately protected against XS-Leaks exploits coming from its child frames.

### Cross origin isolation limits
If crossOriginIsolation is secure, why change the model? Essentially, because it is hard to deploy, which slows down adoption of multi-threading features (SABs, WASM threads) needed to improve web app performance.

#### Deployability
The biggest issue for COI deployment is the cascading requirements that COEP imposes. In order for a page to be crossOriginIsolated, all its frames must enforce COEP. This has proven intractable for a large number of pages.

In order to alleviate the issue, we introduced [credentialless iframes](https://developer.mozilla.org/en-US/docs/Web/Security/IFrame_credentialless). However, credentialless iframes come with a blank ephemeral storage partition. We hoped this drawback would be lessened by 3rd party cookie deprecation (3PCD). In the initial version of 3PCD, third party iframes would not have had access to credentials either, meaning that the delta between loading in a credentialless iframe and a regular iframe would have been small. But 3PCD now includes a sizable number of carve-outs or user driven opt-ins, meaning that it might actually take years for the delta to effectively reduce to a point where using credentialless iframes does not negatively impact functionality.

Finally, credentialless iframes were designed for 3rd party iframes. However, we find that large applications can struggle with getting 1st party iframes to enforce COEP. These iframes might be maintained by other parts of the organization that have different priorities from the top-level frame that want to get access to COI-gated APIs, and might not prioritize supporting COEP. This adds a lot of delay to COI deployment in the top-level frame.

#### Iframes getting access to COI-gated APIs
The crossOriginIsolation model that we built was designed to give access to leaky APIs to the top-level frame (and same-origin frames). Access to COI-gated APIs would then be granted to cross-origin child frames by the top-level frame.

COI was designed in this way because that’s the only model that works without OOPIF. At the same time, use cases that we had for SharedArrayBuffers at the time were all in the top-level frame.

However, since then, various developers have expressed to us the desire to have access to COI-gated APIs in embedded widgets. For example, a chat widget or a 3d map rendering widget might need access to SABs for more efficient calculations, even if the top-level page that embeds it had no interest in SABs itself.

In the current COI model, we have no solution for those use cases. Instead, widget developers would have to maintain two versions of their widgets, with and without SABs, and serve the SAB one only to embedders that deploy crossOriginIsolation. Maintaining two versions of a widget is impractical for most developers.

### Out-of-process iframes evolving landscape
Both of those issues can be solved if we leverage process isolation. We did not do so in the initial COI model because Chrome was the only browser vendor to ship it at the time. However, this is no longer the case.

The other issue was more limited process isolation support on Chrome on Android. However, the APIs gated behind COI are mostly meant for compute heavy websites that are looking for extra computation performance. Developers that want to use SABs are mostly looking at desktop websites, where process isolation is supported. We believe they would be able to navigate the memory cost vs improved computation performance that process isolation on Android implies.

Overall, we believe the time is right to develop a process-isolation based solution for crossOriginIsolation, which will make crossOriginIsolation much more deployable.

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
