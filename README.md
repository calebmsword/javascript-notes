# javascript-notes
Notes on JavaScript.

### Iterators

To create an object which can be iterated over with `for ... of`, you must create a method which returns an iterator.

The method must have the key `Symbol.iterator`. (Note that `Symbol.iterator` is a static property on the `Symbol` class which returns a symbol primitive.) The return value should be an object with a method called `next` that takes zero arguments and returns an object with a `value` and `done` properties.

Example:

```javascript
class Iterable {
  [Symbol.iterator]() {
    const data = Object.values(this);
    let index = -1;

    return {
      next: () => ({ 
        value: data[++index],
        done: !(index in data) 
      })
    };
  }
}

const iterable = new Iterable();
iterable["test"] = "hello";
iterable["test2"] = "world";
for (const property of iterable) console.log(property);  // logs 'hello' and 'world'
```

The object returned by `Iterable[Symbol.iterator]()` is said to conform to the "iterator protocol". The requirements for the protocol are actually less strict that what you would expect, and there are few optional things I did not mention here. See [MDN's documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols) on the iterator protocol for the complete details.

### Monads
A monad consists of three parts. The first is a data structure which wraps a type such as Lists or Optionals. We will call this data structure the "wrapper" associated with the monad. The other two parts are functions which which we will call `wrap` and `transform`.

Let there data types X and Y, where X and Y can be the same. `wrap` and `transform` must have the following signatures:
 1) `wrap`
       - takes an X
       - returns an X-wrapper
 2) `transform`
      - takes an X-wrapper
      - takes a function
         - which takes an X
         - and returns a Y-wrapper
      - returns a Y-wrapper

(Note: The names "wrap" and "transform" are unconventional. In most literature which describes monads, "wrap" is referred to as "return" or "unit", while "transform" is called "bind". The names I provided here are names I made up that makes sense for me.)

Let `wx` be an X-wrapper and let `x` be some X. Let `f` and `g` be functions which can be passed to transform. `wrap` and `transform` must satisfy three constraints:

 1) `transform(wx, wrap) == wrap(x)`
 2) `transform(wrap(x), f) == f(x)`
 3) `transform(wx, x => transform(f(x), g))` `==`
     `transform(transform(wx, f), g)`

JavaScript arrays are monadic wrappers when `wrap` and `transform` are defined as

```javascript
function wrap(x) {
  return [x];
}

/**
 * This function is analagous to Array.prototype.flatMap
 * @example
 * ```
 * const arr = transform(["a", "b"], x => [x, x]);
 * console.log(arr);  // "a", "a", "b", "b"
 * ```
 */
function transform(wx, f) {
  const wy = [];
  wx.forEach(x => {
    f(x).forEach(wy.push));
  }
  return wy;
}
```

Let us implement an Optional monad in JavaScript. We will represent the Optional wrapper as an immutable object with a single "getter" method.

```javascript
function wrap(x) {
  return Object.freeze({
    unwrap: () => x
  });
}

function transform(wx, f) {
  if (wx.unwrap() === null || wx.unwrap() === undefined) {
    return wrap(null);
  }
  
  return f(wx.unwrap());
}

// ------ examples

const wrapper01 = wrap("cheese");
console.log(wrapper01.unwrap())  // "cheese"

const wrapper02 = wrap(null);

function transformer(wrapped) {
  return wrap(`${wrapped}${wrapped}`);
}
console.log(transform(wrapper01, transformer).unwrap());  // "cheesecheese"
console.log(transform(wrapper02, transformer).unwrap());  // null
```

Okay, so monads are a data structure which contains some type and two associated functions `wrap` and `transform`. But when programmers use a monad in JavaScript, one rarely uses the `wrap` and `transform` methods directly. Typically, the designer of the monad will create a single object with convenience methods which compose wrap and transform in useful ways. Let us see how we can do this with an Optional for numeric types. We will represent this monad as a class which contains methods for performing various arithmetic operations on our wrapped number.

```javascript
class NumericOptional {
  #value;
  
  constructor(value) {
    this.#value = value;
    this.add = this.#typeCheckedOperation((value, n) => value + n);
    this.subtract = this.#typeCheckedOperation((value, n) => value - n);
    this.multiply = this.#typeCheckedOperation((value, n) => value * n);
    this.divide = this.#typeCheckedOperation((value, n) => value / n);
    Object.freeze(this);
  }
  
  unwrap() {
    return this.#isProperNumber(this.#value) ? this.#value : null;
  }
  
  transform(f) {
    if (this.unwrap() === null) {
      return NumericOptional.of(null);
    }
    return f(this.unwrap());
  }
  
  static of(value) {
    return new NumericOptional(value);
  }
  
  #isProperNumber(n) {
    return typeof n === "number" && Math.abs(n) !== Infinity && !isNaN(n);
  }

  #typeCheckedOperation(operation) {
    return n => 
      (n instanceof NumericOptional ? n : NumericOptional.of(n)).transform(n => 
        this.transform(value => 
          NumericOptional.of(operation(value, n))));
  }
}

// -------- examples

console.log(
  NumericOptional
    .of(1)
    .add(2)
    .unwrap());  // 3

console.log(
  NumericOptional
    .of(1)
    .add(3)
    .multipy(4)
    .divide(1.25)
    .unwrap());  // 12.8

console.log(
  NumericOptional
    .of(null)
    .add(3)
    .multiply(4)
    .divide(1.25)
    .unwrap()); // null

console.log(
  NumericOptional
    .of(1000.0001)
    .divide(0)
    .unwrap());  // null

console.log(
  NumericOptional
    .of(1)
    .add(3)
    .multiply("4")
    .unwrap());  // null 

console.log(
  NumericOptional
    .of(50)
    .subtract(NaN)
    .unwrap());  // null

const wun = NumericOptional.of(1);
const too = NumericOptional.of(2);
console.log(wun.add(too).unwrap());  // 3
```

Monads allow one to abstract away complicated operations into a declarative sequence of function calls. In the provided example, type checking is abstracted from the user. By utilizing `transform` to perform any operation on the wrapped number, type checking is guaranteed. (Note that `#typeCheckedOperation` is a higher-order function which creates a function that performs `transform` calls.) Also notice that the methods available from our monad are pure. The `#value` private variable is never altered by the class, so all "state changes" are represented by constructions of new `NumericalOptional`s. Predictable state management is another reason to adopt monads.

The "chaining" shown in the previous example is a common way monads are used in JavaScript. However, it is possible to only use compositions of `transform` calls. 

As an exercise, I will also show an implementation of this monad using "this-less" JavaScript. This is the style recommended by Douglas Crockford in "How JavaScript Works".

```javascript
function numericOptionalOf(value) {

  function isProperNumber(n) {
    return typeof n === "number" && Math.abs(n) !== Infinity && !isNaN(n);
  }

  function unwrap() {
    return isProperNumber(value) ? value : null;
  }
  
  function transform(f) {
    if (unwrap() === null) {
      return numericOptionalOf(null);
    }
    return f(unwrap());
  }

  function numericOptionalish(obj) {
    return obj.unwrap === "function" && obj.transform === "function";
  }

  function typeCheckedOperation(operation) {
    return n => 
      numericOptionalOf(numericOptionalish(n) ? n.unwrap() : n).transform(n => 
        transform(value => 
          numericOptionalOf(operation(value, n))));
  }

  function typeCheckedOperationNoArgs(operation) {
    return () => transform(value => numericOptionalOf(operation(value)));
  }
  
  return Object.freeze({
    unwrap,
    transform,
    add: typeCheckedOperation((value, n) => value + n),
    subtract: typeCheckedOperation((value, n) => value - n),
    multiply: typeCheckedOperation((value, n) => value * n),
    divide: typeCheckedOperation((value, n) => value / n)
  });
}

// ----------- examples

console.log(
  numericOptionalOf(1)
    .add(3)
    .multipy(4)
    .divide(1.25)
    .unwrap());  // 12.8

console.log(
  numericOptionalOf(null)
    .add(3)
    .multiply(4)
    .divide(1.25)
    .unwrap()); // null

console.log(
  numericOptionalOf(1000.0001)
    .divide(0)
    .unwrap());  // null

console.log(
  numericOptionalOf(1)
    .add(3)
    .multiply("4")
    .unwrap());  // null 

console.log(
  numericOptionalOf(50)
    .subtract(NaN)
    .unwrap());  // null

const wun = numericOptionalOf(1);
const too = numericOptionalOf(2);
console.log(wun.add(too).unwrap());  // 3
```
