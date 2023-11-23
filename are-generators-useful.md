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

But this is a bad example because thunks solve this problem without introducing the mystical control flow inherent to generators.

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

Another reason you might advocate for generators that is they they allow for *two-way message passing*. Suppose we have a `generator` and then we do `const iterator = generator()`. The function call `iterator.next(value)` sends information in two ways. The return value is an object containing the value yielded from the `generator`. Meanwhile, `value` is passed to `generator` and replaces the most recent `yield <expression>` (or is ignored if it was the first call of `iterator.next`).

This is where I start to think that generators are more interesting. However, it should be understood that this is not anything new. For example, we could create the utility

```javascript
const DONE = Symbol("done");

function taskGenerator(...tasks) {
  let currentTask = 0;

  return (message) => {
    if (currentTask >= tasks.length) return DONE;
    
    const func = tasks[currentTask++];
    return func(message);
  }
}
```

Here is a simple example where an ES6 generator and `taskGenerator` are fully equivalent:

```javascript

// ------ With ES6 generator
function* generator() {
  const cheese = yield "American";
  console.log(cheese);
}

const iterator = generator();

console.log(iterator.next().value);  // "American"
iterator.next("gruyere");  // "gruyere"


// ------ With taskGenerator

const doNextTask = taskGenerator(() = > {
  return "American";
}, (cheeseMessage) => {
  console.log(cheeseMessage);
});

console.log(doNextTask());  // "American"
doNextTask("gruyere");  // "gruyere"
```

So, two-way message passing is not something generators make possible. *Maybe* you could argue that they make it more ergonomic.

Now, there is one behavior of generators that I think is difficult to emulate purely with functions: how one can "throw errors" into generators:

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

console.log(iterator.next().value);
// "gruyere"
console.log(iterator.next("parmesean");
// "cheese is:" "parmesean"
// "rye"
iterator.throw(new Error("oops!"));
// "oops!"
```

If, for some reason, this pattern suits your needs, then generators are probably your best bet.

Most polyfills for generators use `switch` statements to emulate this behavior. The state of the polyfilled generator is represented as a number, and the switch statement is on this number. When you call `next` or `throw`, the generator checks the current state and determines the next number it should be based on whether `next` or `throw` was called. Each case in the switch statement then executes some behavior and then returns an object with `value` and `done` keys. Clearly, generators are much more ergonomic and easy to read here.
