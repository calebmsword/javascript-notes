### Monads
A monad consists of three parts. The first is a data structure which cpntains a type such as Lists or Optionals. We will call this data structure the "container" associated with the monad. The other two parts are functions which which we will call `wrap` and `transform`.

Let there data types X and Y, where X and Y can be the same. `wrap` and `transform` must have the following signatures:
 1) `wrap`
       - takes an X
       - returns an X-container
 2) `transform`
      - takes an X-wrapper
      - takes a function
         - which takes an X
         - and returns a Y-container
      - returns a Y-container

(Note: The names "container", "wrap" and "transform" are unique to these notes. In most literature which describes monads, "wrap" is referred to as "return" or "unit" while "transform" is called "bind".)

Let `wx` be an X-container and let `x` be some X. Let `f` and `g` be functions which can be passed to transform. `wrap` and `transform` must satisfy the "monad laws":

 1) `transform(wx, wrap) == wrap(x)`
 2) `transform(wrap(x), f) == f(x)`
 3) `transform(wx, x => transform(f(x), g)) == transform(transform(wx, f), g)`

JavaScript arrays are monadic containers when `wrap` and `transform` are defined as

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
    f(x).forEach(y => wy.push(y)));
  }
  return wy;
}
```

Let us implement an Optional monad in JavaScript. We will represent the Optional container as an immutable object with a single "getter" method.

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

Okay, so monads are a data structure which contains some type and has two functions `wrap` and `transform` which adhere to the "monad laws". But when programmers use a monad in JavaScript, one rarely uses the `wrap` and `transform` methods directly. Typically, the designer of the monad will create a single object with convenience methods which compose wrap and transform in useful ways. Let us see how we can do this with an Optional for numeric types. We will represent this monad as a class which contains methods for performing various arithmetic operations on our contained number. We'll let the constructor of the method act as the `wrap` function and a `transform` method to the class. 

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

  // Isn't transform supposed to take two arguments (wx, f)?
  // Recall that methods are implicitly passed `this` as an argument
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

The "chaining" shown in the previous example is a common way monads are used in JavaScript. But it should be noted that the `add`, `subtract`, `multiply`, and `divide` methods are just shorthands for compositions of `transform` calls. If you wanted to, all practical usage of a monad can be performed through explicit compositions of `transform` calls.

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
