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

A feature of generators that is not often discussed is that they allow for *two-way message passing*. Suppose we have a `generator` and then we do `const iterator = generator()`:

 - `iterator.next(value)` sends information in two ways. The return value is an object containing the value yielded from the `generator`. Meanwhile, `value` is passed to `generator` and replaces the most recent `yield <expression>` (or is ignored if no `yield` was reach yet).

Of course, we could accomplish something like this without generators:

```javascript
const DONE = Symbol("done");

function taskGenerator() {
  let currentStep = 0;

  const tasks = [
    func1,
    func2,
    func3,
    // etc
  ];

  return (message) => {
    if (index >= tasks.length) return DONE;
    
    const func = tasks[currentStep++];
    return func(message);
  }
}

const doNextTask = taskGenerator();

for(let doNextTask = taskGenerator(), prevReturn;
    prevReturn !== DONE;
    prevReturn = doNextTask(prevReturn)) {}
```

...to be continued
