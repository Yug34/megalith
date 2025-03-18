---
author: Yug Gajjar
pubDatetime: 2025-02-21T15:22:00Z
modDatetime: 2025-02-21T09:12:47.400Z
title: 'Code that tests itself: Asserting your assumptions'
slug: better-testing
featured: true
draft: false
tags:
  - tech
description:
  Better testing
---

### Preface

I've got you with this catchy title, but give me a minute; really quickly, I want to give
you some background on the way we at Atlan fixed our testing troubles.

A feature that we're working on happened to be untestable with
unit tests and E2E test suites; we were developing a Directed Acyclic Graph (DAG)
renderer on the front-end that renders nodes and edges into an HTML `<canvas>` element.

Of course, because we were rendering into the canvas, it was difficult to test
our feature due to `<canvas>`'s lack of DOM representation. 
Screenshot comparison or pixel-based verification is obviously very difficult,
and so is simulating click events because those needed coordinate-based positioning.

So we needed to come up with a way to test if:
- What we were rendering onto the canvas was actually correct. 
- The data we are fetching from the back-end is of the correct form.
- The state of the app on the front-end is correct.
- The assumptions that we are making while developing the feature about our code and it's state are correct.

...so, we made it so the code tests itself.

We threw the tests INTO the code that we wrote, as opposed to running CI workflows for
unit tests or E2E tests.

The outcomes were:

- 

### Enter: Assertions



[//]: # (
    How assertions help with testing <canvas>
    (Code correctness / making sure your assumptions are correct
    (Make sure the state of the data is correct
    (Code is actually tested during the run-time as opposed to a CI workflow
    (Error tracking and monitoring just like Sentry
    (Comments get stale, assertions don't, they will throw
)
