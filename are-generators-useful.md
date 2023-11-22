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

But this is a bad example because closures solve this problem without introducing the mystical control flow inherent to generators.

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

Another reason you might advocate for generators that is they they allow for *two-way message passing*. Suppose we have a `generator` and then we do `const iterator = generator()`. The function call `iterator.next(value)` sends information in two ways. The return value is an object containing the value yielded from the `generator`. Meanwhile, `value` is passed to `generator` and replaces the most recent `yield <expression>` (or is ignored if no `yield` was reach yet).

But I don't think this is convincing either because we could accomplish something like this without generators:

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
    if (currentStep >= tasks.length) return DONE;
    
    const func = tasks[currentStep++];
    return func(message);
  }
}

const doNextTask = taskGenerator();

let prevReturn;
while (prevReturn !== DONE) {
  let message;
  // do something with prevReturn and maybe assign something to message
  prevReturn = doNextTask(message);
}
```

I think the hardest thing to emulate is how one can "throw errors" into generators:

```javascript
function* myGenerator() {
  try {
    const cheese = yield "gruyere";
    console.log("cheese is:", cheese);
    const bread = yield "rye";
    const beer = yield "IPA";
  }
  catch(error) {
    console.log(error.message);
  }
}

const iterator = myGenerator();

console.log(iterator.next());
// "gruyere"
console.log(iterator.next("parmesean");
// "cheese is:" "parmesean"
// "rye"
iterator.throw(new Error("oops!"));
// "oops!"
```

If, for some reason, this pattern suits your needs, then generators are probably preferable to closures for this problem (you still could implement equivalent behavior without generators if you were clever enough).
