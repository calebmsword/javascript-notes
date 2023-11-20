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
