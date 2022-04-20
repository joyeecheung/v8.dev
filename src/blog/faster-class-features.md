---
title: 'Faster initialization of instances with new class features'
author: '[Joyee Cheung](https://twitter.com/JoyeeCheung), instance initializer'
avatars:
  - 'joyee-cheung'
date: 2022-04-14
tags:
  - internals
description: 'Initializations of instances with new class features have become faster since V8 v9.7.'
---

Class fields have been shipped in V8 since v7.2 and private class methods have been shipped since v8.4. After the proposals reached stage 4 in 2021, work had begun to improve the support of the new class features in V8 - until then, there had been two main issues affecting their adoption:

1. The initialization of class fields and private methods was much slower than the assignment of ordinary properties.
2. The class field initializers were broken in [startup snapshots](https://v8.dev/blog/custom-startup-snapshots) used by embedders like Node.js and Deno to speed up the bootstrapping of themselves or user applications.

The first issue has been fixed in V8 v9.7 and the fix for the second issue has been released in V8 v10.0. This post covers how the first issue was fixed, for another read about how the fix of the snapshot issue, check out [this post](https://joyeecheung.github.io/blog/2022/04/14/fixing-snapshot-support-of-class-fields-in-v8/).

## Optimizing class fields

To get rid of the performance gap between the assignment of ordinary properties and the initialization of class fields, we updated the existing [inline cache (IC) system](https://mathiasbynens.be/notes/shapes-ics) to work with the latter. Before v9.7, V8 always used a costly runtime call for class field initializations. With v9.7, when V8 considers the pattern of the initialization to be predictable enough, it uses a new IC to speed up the operation just like what it does for assignments of ordinary properties.

![Performance of initializations, optimized](/_img/faster-class-features/class-fields-performance-optimized.svg)

![Performance of initializations, interpreted](/_img/faster-class-features/class-fields-performance-interpreted.svg)

### The original implementation of class fields

To implement private fields, V8 makes use of the internal private symbols &mdash; they are an internal V8 data structure similar to standard `Symbol`s, except not enumerable when used as a property key. Take this class for an example:


```js
class A {
  #a = 0;
  b = this.#a;
}
```

V8 would collect the class field initializers (`#a = 0` and `b = this.#a`) and generate a synthetic instance member function with the initializers as the function body. The bytecode generated for this synthetic function used to be something like this:

```cpp
// Load the private name symbol for `#a` into r1
LdaImmutableCurrentContextSlot [2]
Star r1

// Load 0 into r2
LdaZero
Star r2

// Move the target into r0
Mov <this>, r0

// Use the %AddPrivateField() runtime function to store 0 as the value of
// the property keyed by the private name symbol `#a` in the instance,
// that is, `#a = 0`.
CallRuntime [AddPrivateField], r0-r2

// Load the property name `b` into r1
LdaConstant [0]
Star r1

// Load the private name symbol for `#a`
LdaImmutableCurrentContextSlot [2]

// Load the value of the property keyed by `#a` from the instance into r2
LdaKeyedProperty <this>, [0]
Star r2

// Move the target into r0
Mov <this>, r0

// Use the %CreateDataProperty() runtime function to store the property keyed
// by `#a` as the value of the property keyed by `b`, that is, `b = this.#a`
CallRuntime [CreateDataProperty], r0-r2
```

Compare the class in the previous snippet to a class like this:

```js
class A {
  constructor() {
    this._a = 0;
    this.b = this._a;
  }
}
```

Technically these two classes are not equivalent, even ignoring the difference in visibility between `this.#a` and `this._a`. The specification mandates "define" semantics instead of "set" semantics. That is, the initialization of class fields does not trigger setters or `set` Proxy traps. So an approximation of the first class should use `Object.defineProperty()` instead of simple assignments to initialize the properties. In addition, it should throw if the private field already exists in the instance (in case the target being initialized is overridden in the base constructor to be another instance):

```js
class A {
  constructor() {
    // What the %AddPrivateField() call roughly translates to:
    const _a = %PrivateSymbol('#a')
    if (_a in this) {
      throw TypeError('Cannot initialize #a twice on the same object');
    }
    Object.defineProperty(this, _a, {
      writable: true,
      configurable: false,
      enumerable: false,
      value: 0
    });
    // What the %CreateDataProperty() call roughly translates to:
    Object.defineProperty(this, 'b', {
      writable: true,
      configurable: true,
      enumerable: true,
      value: this[_a]
    });
  }
}
```

To implement the specified semantics before the proposal finalized, V8 used calls to runtime functions since they are more flexible. As shown in the bytecode above, the initialization of public fields was implemented with `%CreateDataProperty()` runtime calls, while the initialization of private fields was implemented with `%AddPrivateField()`. Since calling into the runtime incurs a significant overhead, the initialization of class fields was much slower compared to the assignment of ordinary object properties.

In most use cases, however, the semantic differences are insignificant. It would be nice to have the performance of the optimized assignments of properties in these cases &mdash; so a more optimal implementation was created after the proposal finalized.

### Optimizing private class fields and computed public class fields

To speed up initialization of private class fields and computed public class fields, the implementation introduced a new machinery to plug into the [inline cache (IC) system](https://mathiasbynens.be/notes/shapes-ics) when handling these operations. This new machinery comes in three cooperating pieces:

- In the bytecode generator, a new bytecode `DefineKeyedOwnProperty`. This gets emitted when generating code for the `ClassLiteral::Property` AST nodes representing class field initializers.
- In the TurboFan JIT, a corresponding IR opcode `JSDefineKeyedOwnProperty`, which can be compiled from the new bytecode.
- In the IC system, a new `DefineKeyedOwnIC` that is used in the interpreter handler of the new bytecode as well as the code compiled from the new IR opcode. To simplify the implementation, the new IC reuses some of the code in `KeyedStoreIC` which was intended for ordinary property stores.

Now when V8 encounters this class:

```js
class A {
  #a = 0;
}
```

It generates the following bytecode for the initializer `#a = 0`:

```cpp
// Load the private name symbol for `#a` into r1
LdaImmutableCurrentContextSlot [2]
Star0

// Use the DefineKeyedOwnProperty bytecode to store 0 as the value of
// the property keyed by the private name symbol `#a` in the instance,
// that is, `#a = 0`.
LdaZero
DefineKeyedOwnProperty <this>, r0, [0]
```

When the initializer is executed enough times, V8 allocates one [feedback vector slot](https://ponyfoo.com/articles/an-introduction-to-speculative-optimization-in-v8) for each field being initialized. The slot contains the key of the field being added (in the case of the private field, the private name symbol) and a pair of [hidden classes](https://v8.dev/docs/hidden-classes) between which the instance has been transitioning as the result of field initialization. In subsequent initializations, the IC uses the feedback to see if the fields are initialized in the same order on instances with the same hidden classes. If the initialization matches the pattern that V8 has seen before (which is usually the case), V8 takes the fast path and performs the initialization with pre-generated code instead of calling into the runtime, thus speeding up the operation. If the initialization does not match a pattern that V8 has seen before, it falls back to a runtime call to deal with the slow cases.

### Optimizing named public class fields

To speed up initialization of named public class fields, we reused the existing `DefineNamedOwnProperty` bytecode which calls into `DefineNamedOwnIC` either in the interpreter or through the code compiled from the `JSDefineNamedOwnProperty` IR opcode.

Now when V8 encounters this class:

```js
class A {
  #a = 0;
  b = this.#a;
}
```

It generates the following bytecode for the `b = this.#a` initializer:

```cpp
// Load the private name symbol for `#a`
LdaImmutableCurrentContextSlot [2]

// Load the value of the property keyed by `#a` from the instance into r2
// Note: LdaKeyedProperty is renamed to GetKeyedProperty in the refactoring
GetKeyedProperty <this>, [2]

// Use the DefineKeyedOwnProperty bytecode to store the property keyed
// by `#a` as the value of the property keyed by `b`, that is, `b = this.#a;`
DefineNamedOwnProperty <this>, [0], [4]
```

The original `DefineNamedOwnIC` machinery could not be simply plugged into the handling of the named public class fields, since it was originally intended only for object literal initialization. Previously it expected the target being initialized to be an object that has not yet been touched by the user since its creation, which was always true for object literals, but the class fields can be initialized on user-defined objects when the class extends a base class whose constructor overrides the target:

```js
class A {
  constructor() {
    return new Proxy(
      { a: 1 },
      {
        defineProperty(object, key, desc) {
          console.log('object:', object);
          console.log('key:', key);
          console.log('desc:', desc);
          return true;
        }
      });
  }
}

class B extends A {
  a = 2;
  #b = 3;  // Not observable.
}

// object: { a: 1 },
// key: 'a',
// desc: {value: 2, writable: true, enumerable: true, configurable: true}
new B();
```

To deal with these targets, we patched the IC to fall back to the runtime when it sees that the object being initialized is a proxy, if the field being defined already exists on the object, or if the object just has a hidden class that the IC has not seen before. It is still possible to optimize the edge cases if they become common enough, but so far it seems better to trade the performance of them for simplicity of the implementation.

## Optimizing private methods

### The implementation of private methods

In [the specification](https://tc39.es/ecma262/#sec-privatemethodoraccessoradd), the private methods are described as if they are installed on the instances but not on the class. In order to save memory, however, V8's implementation stores the private methods along with a private brand symbol in a context associated with the class. When the constructor is invoked, V8 only stores a reference to that context in the instance, with the private brand symbol as the key.

![Evaluation and instantiation of classes with private methods](/_img/faster-class-features/class-evaluation-and-instantiation.svg)

When the private methods are accessed, V8 walks the context chain starting from the execution context to find the class context, reads a statically known slot from the found context to get the private brand symbol for the class, then checks if the instance has a property keyed by this brand symbol to see if the instance is created from this class. If the brand check passes, V8 loads the private method from another known slot in the same context and completes the access.

![Access of private methods](/_img/faster-class-features/access-private-methods.svg)

Take this snippet as an example:

```js
class A {
  #a() {}
}
```

V8 used to generate the following bytecode for the constructor of `A`:

```cpp
// Load the private brand symbol for class A from the context
// and store it into r1.
LdaImmutableCurrentContextSlot [3]
Star r1

// Load the target into r0.
Mov <this>, r0
// Load the current context into r2.
Mov <context>, r2
// Call the runtime %AddPrivateBrand() function to store the context in
// the instance with the private brand as key.
CallRuntime [AddPrivateBrand], r0-r2
```

Since there was also a call to the runtime function `%AddPrivateBrand()`, the overhead made the constructor much slower than constructors of classes with only public methods.

### Optimizing initialization of private brands

To speed up the installation of the private brands, in most cases we just reuse the `DefineKeyedOwnProperty` machinery added for the optimization of private fields:

```cpp
// Load the private brand symbol for class A from the context
// and store it into r1
LdaImmutableCurrentContextSlot [3]
Star0

// Use the DefineKeyedOwnProperty bytecode to store the
// context in the instance with the private brand as key
Ldar <context>
DefineKeyedOwnProperty <this>, r0, [0]
```

![Performance of instance initializations of classes with different methods](/_img/faster-class-features/private-methods-performance.svg)

There is a caveat, however: if the class is a derived class whose constructor calls `super()`,  the initialization of the private methods - and in our case, the installation of the private brand symbol - has to happen after `super()` returns:

```js
class A {
  constructor() {
    // This throws from a new B() call because super() has not yet returned.
    this.callMethod();
  }
}

class B extends A {
  #method() {}
  callMethod() { return this.#method(); }
  constructor(o) {
    super();
  }
};
```

As described before, when initializing the brand, V8 also stores a reference to the class context in the instance. This reference isn't used in brand checks, but is instead intended for the debugger to retrieve a list of private methods from the instance without knowing which class it is constructed from. When `super()` is invoked directly in the constructor, V8 can simply load the context from the context register (which is what `Mov <context>, r2` or `Ldar <context>` in the bytecodes above does) to perform the initialization, but `super()` can also be invoked from a nested arrow function, which in turn can be invoked from a different context. In this case, V8 falls back to a runtime function (still named `%AddPrivateBrand()`) to look for the class context in the context chain instead of relying on the context register. For example, for the `callSuper` function below:

```js
class A extends class {} {
  #method() {}
  constructor(run) {
    const callSuper = () => super();
    // ...do something
    run(callSuper)
  }
};

new A((fn) => fn());
```

V8 now generates the following bytecode:

```cpp
// Invoke the super constructor to construct the instance
// and store it into r3.
...

// Load the private brand symbol from the class context at
// depth 1 from the current context and store it into r4
LdaImmutableContextSlot <context>, [3], [1]
Star4

// Load the depth 1 as an Smi into r6
LdaSmi [1]
Star6

// Load the current context into r5
Mov <context>, r5

// Use the %AddPrivateBrand() to locate the class context at
// depth 1 from the current context and store it in the instance
// with the private brand symbol as key
CallRuntime [AddPrivateBrand], r3-r6
```

In this case the cost of the runtime call is back so initializing instances of this class is still going to be slower compared to initializing instances of classes with only public methods. It is possible to use a dedicated bytecode to implement what `%AddPrivateBrand()` does, but since invoking `super()` in a nested arrow function is quite rare, we again traded the performance for simplicity of the implementation.

## Final notes

The work mentioned in this blog post is also included in the [Node.js 18.0.0 release](https://nodejs.org/en/blog/announcements/v18-release-announce/). Previously, Node.js switched to symbol properties in a few built-in classes that had been using private fields in order to include them into the embedded bootstrap snapshot as well as to improve the performance of the constructors (see [this blog post](https://www.nearform.com/blog/node-js-and-the-struggles-of-being-an-eventtarget/) for more context). With the improved support of class features in V8, Node.js [switched back to private class fields](https://github.com/nodejs/node/pull/42361) in these classes and Node.js's benchmarks showed that [these changes did not introduce any performance regressions](https://github.com/nodejs/node/pull/42361#issuecomment-1068961385).

Thanks to Igalia and Bloomberg for contributing this implementation!
