Many examples I see advocating for ES6 generators include something like the following:

```javascript
function uniqueIdGenerator() {
  let i = 0;
  while(true) yield i++;
}

const uniqueIdIterator = uniqueIdGenerator();

console.log(uniqueIdIterator.next().value);  // 1
console.log(uniqueIdIterator.next().value);  // 2
console.log(uniqueIdIterator.next().value);  // 3
```

But this is a bad example because closures solve this problem without introducing the mystical control flow generators require.

```javascript
function getUniqueIdFactory() {
  let i = 0;
  return () => i++;
}

const getUniqueId = getUniqueIdFactory();

console.log(getUniqueId());  // 1
console.log(getUniqueId());  // 2
console.log(getUniqueId());  // 3
```

The control flow of generators is reminiscent of goto-label syntax, which is never a good thing. This alone makes it hard to recommend them.

What I wish people would advocate is how generators allow for *two-way message passing*. Suppose we have a `generator` and then we do `const iterator = generator()`:

 - `iterator.next(value)` sends information in two ways. The return value is an object containing the value yielded from the `generator`. Meanwhile, `value` is passed to `generator` and replaces the most recent `yield <expression>` (or is ignored if no `yield` was reach yet).

I am not sure this a good enough reason to justify using generators, but it is the only one that I find convincing.
