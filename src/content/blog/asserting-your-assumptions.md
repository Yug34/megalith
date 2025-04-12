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
  How we tested an un-testable feature, without touching unit tests!
---

# Preface

I've got you with this catchy title, but gimme a minute; really quickly, I want to give
you some background on the way we at Atlan fixed our testing troubles.

A feature that we were working on happened to be un-testable with
unit tests and E2E test suites; we were developing a Directed Graph (Digraph)
renderer on the front-end that paints nodes and edges into an HTML `<canvas>` element. It looked
something like this:

<br />

![Lineage](../../assets/images/lineage.png)
<p style="width: 100%; text-align: center;">
Data Lineage - The digraph renderer
</p>

<br />

Since we were rendering into the canvas, it was difficult to test
our feature due to `<canvas>`'s lack of DOM representation.
Screenshot comparison or pixel-based verification is difficult,
and so is simulating click events because that needs coordinate-based positioning.

**...so, we made it so the code tests itself.**

We threw the tests INTO the code that we wrote, as opposed to running CI workflows for
unit tests or E2E tests. This also means that the tests run during the app's run-time.

The outcomes were:
- We were able to test if what we rendered onto the canvas was actually correct. 
- Make sure that the data we are fetching from the back-end is of the correct form.
- The state of the front-end app is correct and is never something we don't expect.
- The assumptions that we make about our code and it's state are correct.

We essentially defined a set of "allowed behaviours" that the app should operate in.
And any time the app does something that we don't expect, we know exactly what to fix.
We defined the "happy path" and made sure we know when the app diverges from it.

<hr />

### ...But how did we do that?

# Enter: Assertions

When we write code, we tend to make reasonable assumptions about the code and
the environment we are writing the code in.

These assumptions could be about the state of your system, the data within the system and how it is passed around,
what functions expect as input and are expected to output, their interfacing, and so on.

Sometimes, these assumptions are wrong and this results in developer errors. Hell, sometimes even when you
are aware about your environment, the code you write is still very prone to developer errors.

Like this well-known behaviour in JavaScript:

<br />

![JavaScript bug](../../assets/images/js.png)

<br />

To no fault of the developer, they would incorrectly assume `0.1 + 0.2` to be `0.3`.
Because why would anyone ever assume otherwise?

Assertions are a way to enforce correctness into your code such that your code itself screams
at you when it is doing something you didn't expect it to, or when it is working with something you didn't
expect it to.

This also results in a subtle difference in the way you think of testing:
- With unit tests, you manually predict and account for edge cases and make sure they are handled properly.
- With assertions, you define the happy path of the software directly, and anytime the code diverges from it, it will automatically complain.

<br />

We'll move on to much more powerful examples of assertions, but let's start with
some really simple assertions just using `console.assert`, before we get to the heart of it:

```ts
type Foo = {
  bar?: any // Peak production TypeScript code
}

const factorialFoo = (foo: Foo) => {
  console.assert(Number.isInteger(foo.bar), "bar must exist and must be an integer")
  console.assert(foo.bar > 0, "bar must be positive")

  // We now proceed with the confidence that our assumptions are true
  return factorial(foo.bar)
}

// Every time you use the function in an unintended way, the assertion fails.
factorialFoo({ bar: '5.2' })    // Assertion failed: bar must exist and must be an integer
factorialFoo({})                // Assertion failed: bar must exist and must be an integer
factorialFoo({ bar: -5 })       // Assertion failed: bar must be positive

// But with valid input, it works just as expected:
factorialFoo({ bar: 5 })        // 120
```

So what just happened here?

We wrote two assertions within our function definition that state exactly
what conditions must be true for our function to work as expected:

`"bar must exist and must be an integer"` and `"bar must be positive or 0"`.

Any time these assumptions are false, `console.assert` will tell you that the assertion is failing.

We have essentially programmed in an invariance; that `Foo.bar` should always be a number.

### **Instead of making safeguards for all that `foo.bar` could be, we asserted exactly what it _should be_**.

But this isn't yet using assertions to their full capacity.

<br />

# Levelling up our asserts

So far in this blog I've shown you just a basic use case of assertions with `console.assert`, but that's just the
building block of an `assert()` implementation you would actually use.

This `assert` implementation has the following features:
- Control when the assertions run:
  - If the `env.MODE` is `development`, enable all the assertions.
  - If the `env.MODE` is `production`, enable only the assertions intended to run in production.
  - If the `env.MODE` is `noAssert`, disable all the assertions.
- Tracking when an assertion fails and logging it with the program's state for Sentry-like error monitoring.
- Ability to debug the program's seed state that lead to an assertion failing.
- Ability to throw an error and crash the program if a failing assertion might lead to a critical bug.

Here's a simple factory function (`createAssert`) in just `~50` lines of code that produces assertion functions:

<br />

```typescript
const env = import.meta.env.MODE

const trackEvent = (eventName: string, data: any) => {
  // Send event to Sentry or other monitoring services
}

type AssertFunction = (assertion: string, condition: boolean, programState?: any) => void

interface AssertOptions {
  trackInProd?: boolean
  throwOnFail?: boolean
}

// Factory function to create assertion functions
const createAssert = (options: AssertOptions = {}): AssertFunction => {
  const { trackInProd = false, throwOnFail = false } = options

  if (env === 'noAssert') {
    return () => {}
  }
  
  if (env === 'development') {
    return (assertion: string, condition: boolean, programState?: any) => {
      if (!condition) {
        console.error(`Assertion Failed: ${assertion}`)
        console.error('Program state:', programState ?? {})
        
        if (throwOnFail) {
          const error = new Error(`Assertion failed: ${assertion}`)
          throw error
        }
      }
    };
  }
  
  if (env === 'production' && trackInProd) {
    return (assertion: string, condition: boolean, programState?: any) => {
      if (!condition) {
        trackEvent(
          `assertion_failed_${assertion}`,
          { programState: programState ?? {} }
        );

        if (throwOnFail) {
          const error = new Error(`Assertion failed: ${assertion}`);
          throw error
        }
      }
    }
  }
  
  // Default for other envs
  return () => {}
};
```

<br />

With the factory function implemented, this is how you create the assert functions:

```ts
// Runs only in dev mode, disabled in prod:
const assert = createAssert() 
// Runs only in dev mode, and throws when it fails:
const criticalAssert = createAssert({ throwOnFail: true }) 
// Tracks failures in prod and logs it to Sentry/Mixpanel
const prodAssert = createAssert({ trackInProd: true })
// Tracks failures in prod and throws
const criticalProdAssert = createAssert({ throwOnFail: true, trackInProd: true })
```

<br />

And this is how you use them for actual code:


```ts
// In a funds transfer service
async function transferFunds(sourceAccountId, destinationAccountId, amount, userId) {
  // Validate inputs
  criticalProdAssert(
    "source_account_exists",
    typeof sourceAccountId === 'string' && sourceAccountId.length > 0,
    { sourceAccountId }
  );
  
  criticalProdAssert(
    "destination_account_exists",
    typeof destinationAccountId === 'string' && destinationAccountId.length > 0,
    { destinationAccountId }
  );
  
  criticalProdAssert(
    "amount_valid",
    typeof amount === 'number' && amount > 0 && isFinite(amount),
    { amount }
  );
  
  // Load accounts:
  const sourceAccount = await accountService.getAccount(sourceAccountId);
  const destinationAccount = await accountService.getAccount(destinationAccountId);
  
  criticalProdAssert(
    "source_account_found",
    sourceAccount !== null,
    { sourceAccountId, userId }
  );
  
  criticalProdAssert(
    "destination_account_found",
    destinationAccount !== null,
    { destinationAccountId, userId }
  );
  
  criticalProdAssert(
    "source_account_belongs_to_user",
    sourceAccount.ownerId === userId,
    { 
      sourceAccountId, 
      sourceAccountOwnerId: sourceAccount.ownerId, 
      requestingUserId: userId 
    }
  );
  
  criticalProdAssert(
    "sufficient_funds",
    sourceAccount.balance >= amount,
    { 
      sourceAccountId,
      currentBalance: sourceAccount.balance,
      requestedAmount: amount,
    }
  );
  
  // All assertions pass! We are safe to proceed:
  const transaction = await db.beginTransaction();
  
  try {
    await accountService.debitAccount(sourceAccountId, amount, transaction);
    await accountService.creditAccount(destinationAccountId, amount, transaction);
    
    // Verify transaction integrity - critical to financial consistency
    const updatedSourceAccount = await accountService.getAccount(sourceAccountId, transaction);
    criticalProdAssert(
      "debit_applied_correctly",
      Math.abs(sourceAccount.balance - updatedSourceAccount.balance - amount) < 0.001,
      {
        sourceAccountId,
        originalBalance: sourceAccount.balance,
        newBalance: updatedSourceAccount.balance,
        amount,
        difference: sourceAccount.balance - updatedSourceAccount.balance
      }
    );
    
    // All is good, commit the transaction!
    await transaction.commit();
    return {
      success: true,
      transactionId: transaction.id,
      newSourceBalance: updatedSourceAccount.balance
    };
  } catch (error) {
    // Some assertion failed, rollback on thrown error
    await transaction.rollback();
    throw error;
  }
}
```

Every assertion you apply here acts as documentation about what the environment assumes is an invariant
that must hold true, and also allows you to move forward with the confidence that your assumptions are correct.



<!-- ```
Statically typing my English natural language queries so the AI doesn't misinterpret them:

interface Statement {
  subject: string
  action: string
  object?: string
  intent: "sarcastic" | "serious" | "joke" | "request" | "question"
  tone?: "passive_aggressive" | "enthusiastic" | "deadpan" | "condescending"
  emotion?: "angry" | "happy" | "confused" | "apathetic" | "delirious"
  formality?: "casual" | "formal" | "corporate"
}

This clarified my prompts by a lot, so here's the logical next step:

What if I create a natural language such that I could talk to the computer directly in machine code after compiling the natural language phrases.

Hi, I'm Guido Van Rossum.
``` -->