---
author: Yug Gajjar
pubDatetime: 2023-10-15T09:22:00Z
modDatetime: 2023-10-15T09:12:47.400Z
title: "YDKTS: Nominal Types"
slug: ydkts-nominal-types
featured: true
draft: false
tags:
  - tech
description:
  TypeScript gives you a false sense of security in type-safety, by only checking type structures. We fix that with a Nominal type system; allowing each type to have an identity beyond just it's member's types.
---

## What's the problem with TypeScript?

TypeScript is a **structural** type system; meaning if you compare two types in TypeScript that have the same members, the TypeScript compiler will consider them equivalent.

As a result of structural type-checking, two types with the same structure will be interchangeable. Here's an example:

```typescript
type PostId = string
type UserId = string

let userId: UserId = 'abc123'
let postId: PostId = 'xyz789'

userId = postId // <-- This should NOT work, but TS sees no problem in it.
```

This leads to a false sense of security; the compiler considering types with the same structure equivalent leads to some hard-to-find bugs.

Here's a simple example of it:

```typescript
let userId: UserId = 'abc123'
let postId: PostId = 'xyz789'
const fetchPostById = (postId: PostId): Post => {
    // ...
}

fetchPostById(userId) // Whoops. TS won't catch this -- that's dangerous.
```

## Okay, now what?

Casually let your back-end figure out what to do with the given `userId` where it should've received a `postId`?

No, let me help you avoid digging into the codebase on a Saturday afternoon because some feature you shipped yesterday is broken:

## * Enter **_✨ `Nominal Types` ✨_** *

It's just 5 lines of code and really easy to add to your projects, so don't install a package for this, ThePrimeagen will hate you for it:

```typescript
declare const __type: unique symbol

export type Nominal<Identifier, Type> = Type & {
    readonly [__type]: Identifier
}
```

Import `Nominal` and use it as such:

```typescript
import { Nominal } from "./nominal"

type UserId = Nominal<'UserId', string>
type PostId = Nominal<'PostId', string>

let userId = 'randomId' as UserId
let postId = 'randomId' as PostId

userId = postId // <-- TS will catch this, and won't allow it! :)

// TS2322: Type PostId is not assignable to type UserId
// Type PostId is not assignable to type { readonly [__type]: 'UserId' }
```

Finally, now the variables in your projects are compatible if and only if they have the exact same types.

## But let's take this a step further...

Now your types aren't just structural and have a real identity other than the members contained in the type. The type itself makes the variables unique.

**_Due to this, now the variable holds some meaning not just in its value, but also in its type_**, and you can take advantage of the variable's type having some meaning throughout your code.

For instance:

```typescript
type SortedArray<T> = Nominal<'SortedArray', Array<T>>

// Sort function that outputs an Array<T> as a SortedArray<T>
const sortArray = <T>(arr: Array<T>): SortedArray<T> => {
    return arr.sort() as SortedArray<T>
}

const binarySearch = <T>(
    sorted: SortedArray<T>,
    val: T
): number | undefined => {
    /* Binary Search here... */
}

const arr = [10, 68, 35, 26, 42, 5]
binarySearch(arr, 39) // This won't work!
// Error: Argument type number[] is not assignable to parameter type SortedArray<number>

const sortedArray = sortArray(arr)
binarySearch(sortedArray, 39) // This will work! :)
```

The pre-requisite of the Binary Search algorithm is that the input array that it operates on should be sorted.

So when you use the Binary Search algorithm, you're going to want to make sure the input array you pass in is always sorted.

To do this, you will perhaps sort the input array within the `binarySearch` function itself... or perhaps make sure to **_always_** sort the array before you pass it to the `binarySearch` function?

With using types this way, you can eliminate so many redundancies in your code that perform such validation checks.

**_This helps you beyond just making your code type-safe; it makes your code more performant._**

Now you **_know for sure_** whether an Array is sorted or not, and handle things accordingly.

## But... how does it work though?

Under the hood, we're taking advantages of JavaScript's `Symbol`:

```typescript
const s1 = Symbol()
const s2 = Symbol()

const s3 = Symbol('randomSymbol')
const s4 = Symbol('randomSymbol')

s1 === s2 // false
s3 === s4 // false
```

`Symbol()` in JavaScript returns a primitive data type (`symbol`) which is guaranteed to be a unique value. The `Symbol` constructor also takes in a string value that serves as a description of that symbol -- note that it has nothing to do with the "identity" of a symbol other than that.

We use symbols as `unique symbol`s, as such:

```typescript
declare const __type: unique symbol

export type Nominal<Identifier, Type> = Type & {
    readonly [__type]: Identifier
}
```

According to the TypeScript Docs; _"Each reference to a unique symbol implies a completely unique identity that’s tied to a given declaration."_

As evident from the TypeScript errors in the first example of Nominal types in practice:

```typescript
// TS2322: Type PostId is not assignable to type UserId
// Type PostId is not assignable to type { readonly [__type]: 'UserId' }
```

All we are really doing is intersecting the type string with `{readonly __type: 'PostId'}` by using TypeScript's intersection type operator `&`.

## Typecasting nominal types

We can't assign the nominal types to variables directly, so we need to use `as` everywhere;

```typescript
const userId: UserId = 'abc123'
// TS2322: Type string is not assignable to type UserId
```

But as a work-around to that, you can have functions to type-cast your variables into nominal types, as such:

```typescript
function UserId(id: string): UserId {
    // optionally add any validation logic
    return id as UserId
}

function PostId(id: string): PostId {
    return id as PostId
}

let userId = UserId('id1')
let postId = PostId('id2')
```

And with just that, `Achievement Unlocked!: Type-safety in TypeScript`.

Cheers! 🥂