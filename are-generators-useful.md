I find it difficult to motivate ES6 generators because they don't add anything new. Instead, they were designed make some things more ergonomic than they used to be.

For example, many examples I see advocating for ES6 generators suggest something like the following:

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

To me, this isn't a very compelling example because thunks already can create simple iterators.

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
const doNextTask = taskGenerator(
  () => "American",
  console.log
);

console.log(doNextTask());  // "American"
doNextTask("gruyere");  // "gruyere"
```

So, two-way message passing is already possible without generators. But some programmers may find their use more ergonomic for this case.

Now, there is one behavior of generators that is difficult to emulate purely with functions: how one can "throw errors" into generators:

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

Most polyfills for generators implement a state machine with `switch` statements to emulate this behavior. The state of the polyfilled generator is represented as a number. When you call `next` or `throw`, the generator checks the current state and determines the next number it should be based on whether `next` or `throw` was called. Each case in the switch statement then executes some behavior, possibly updates the current state, and then returns an object with `value` and `done` keys.

So, while this behavior is strictly possible, it starts to get complicated to emulate with traditional JavaScript functions. If you find a good use case for `try catch` patterns in a generator, it's probably best done with a generator.

You should know that that the iterators returned by generators also have a `return` method. See [MDN's documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator/return) on the topic. The state machine approach can also polyfill this feature.

### Don't use the iterator directly

In general, I think generators make the most sense when you abstract away direct usage of the iterator. For example, you can pass iterators into `for of` loops. An as impractically simple example for the sake of introducing syntax, we will show this using a numeric iterator.

```javascript
function* integerGenerator() {
  let i = 0;
  while (i < 3) yield i++;
}

for (const n of integerGenerator()) console.log(n);
// 0
// 1
// 2
```

Also, when you need to get a sequence of values that are each "processed" in a certain way, generators are well-suited to the process. Yield the value to be processed, process it, and then pass it back to the generator with the iterator. Before `async-await` syntax was introduced into JavaScript, people used very similar syntax using generators. They would yield a Promise and call `iterator.next` with the resolved value in a `then` attached to the Promise. See my notes on async-await (async-await.md) for an example.

There is a [proposal](https://github.com/tc39/proposal-iterator-helpers) for helper methods for iterators that are similar to array methods found in `Array.prototype`. This proposal is in "stage 3" which is the final stage meaning that it is very likely it will be introduced into the language soon. These methods will make generator usage more syntactically concise, although it wouldn't be difficult to implement these helper methods in your own personal utility in the meantime. For example, a mapper could be implemented with 

```javascript
function map(iterator, mapper) {
  return (function* () {
    let value;
    let done = false;
    while (!done) {
      ({ value, done } = iterator.next());
      yield mapper(value);
    }
  })();
}

function* naturals(limit) {
  let i = 0;
  while (i < limit) yield i++;
}

const mapped = map(naturals(10),  x => x * x);

for (const n of mapped) console.log(n);
```

### In conclusion
I don't think you need to use generators. They don't add anything new. Their syntax is strange and their control flow can be hard to follow. 

But I also don't think it is bad to use generators. I think they are best used when you abstract away direct usage of the iterator. Use a `for ... of` loop, wait for iterator helpers to be introduced to JavaScript, or implement some utility which processes the iterator for you.
