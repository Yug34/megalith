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

A feature that we're working on happened to be un-testable with
Vitest unit tests and Playwright E2E test suites; we were developing a Directed Acyclic Graph (DAG)
renderer on the front-end that renders nodes and edges into an HTML `<canvas>` element.

Because we were rendering into the canvas, it was difficult to test
our feature due to `<canvas>`'s lack of DOM representation.
Screenshot comparison or pixel-based verification is difficult,
so is simulating click events because that needs coordinate-based positioning.

...so, we made it so the code tests itself.

We threw the tests INTO the code that we wrote, as opposed to running CI workflows for
unit tests or E2E tests. This also means that the tests run during the app's run-time.

The outcomes were:
- We were able to test if what we rendered onto the canvas was actually correct. 
- Make sure that the data we are fetching from the back-end is of the correct form.
- The state of the front-end app is correct and is never something we don't expect.
- The assumptions that we make about our code and it's state are correct.

We essentially defined a set of "allowed behaviours" that the app should operate in.
And any time the app does something that we don't expect, we know exactly what to fix.

So, how did we do that?

### Enter: Assertions

When we write code, we tend to make reasonable assumptions about the code and
the environment we are writing the code in.

These assumptions could be about the state of your system, the data within the system and how it permeates through,
what functions expect as input and are expected to output, their interfacing, and so on.

Sometimes, these assumptions are wrong and this results in developer errors.

Assertions are a way to enforce correctness into your code, and do it such that your code itself screams
at you when it is doing something you didn't expect it to, or when it is working with something you did
not expect it to.

This also results in a subtle difference in the way that you think of testing:
- With unit tests, you think about, and account for edge cases and make sure they are handled properly.
- With assertions, you define the happy path of the software directly, and anytime the code diverges from it, it will automatically complain.

Here's a very simple example of an assertion in action:

```ts
type Foo = {
  bar?: number
}

const morphFoo = (foo: Foo) => {
  console.assert(foo.bar !== undefined, "foo must exist")
  return foo.bar * 5
}
```

```ts
const env = import.meta.env.MODE
const isDev = env === 'development'

const noOp = () => {}

const createAssertFn = (shouldRun) => (shouldRun ? console.assert : noOp)

const createTrackErrorFn = (shouldRun) =>
    shouldRun
        ? (condition, ...args) => {
              if (!condition) {
                  trackEvent('assertion_failed', {
                      programState: args,
                  })
              }
          }
        : noOp

export const useAssert = () => {
    return {
        assertFnDev: createAssertFn(isDev),
        assertFn: createAssertFn(!isDev),
        assertTrackErrorDev: createTrackErrorFn(isDev),
        assertTrackError: createTrackErrorFn(!isDev),
    }
}
```