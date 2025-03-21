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

[//]: # (
    How assertions help with testing <canvas>
    (Code correctness / making sure your assumptions are correct
    (Make sure the state of the data is correct
    (Code is actually tested during the run-time as opposed to a CI workflow
    (Error tracking and monitoring just like Sentry
    (Comments get stale, assertions don't, they will throw
)


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

// eslint-disable-next-line @typescript-eslint/no-empty-function
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