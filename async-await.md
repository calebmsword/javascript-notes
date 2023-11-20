### async-await syntax is just generator function magic

Consider the following:

```javascript
function getAles() {
  // this is an actual free, public API
  return fetch("https://api.sampleapis.com/beers/ale");
}

async function showFirstAle() {
  try {
    const res = await getAles();
    const json = await res.json();
    console.log(json[0]);
    return "success";
  }
  catch(error) {
    console.log(error);
    return "failure";
  }
}

showFirstAle().then(console.log);  // "success"
```

This function is fully equivalent to a generator function given that we create a utility function which uses the generator function. Let us call the utility function `run`. Before explaining what `run` is or how it is implemented, let us see what the equivalent code would be:

```javascript
function getAles() {
  // this is an actual free, public API
  return fetch("https://api.sampleapis.com/beers/ale");
}

function* showFirstAle() {
  try {
    const res = yield getAles();
    const json = yield res.json();
    console.log(json[0]);
    return "success";
  }
  catch(error) {
    console.log(error);
    return "failure";
  }
}

run(showFirstAle).then(console.log);  // "success"
```

The `async` function is now a generator function. All `await`s are replaced with `yield`s. And instead of calling the generator function, we pass it to `run` which magically uses the generator function in some way.

Before showing an implementation of `run`, let's think about what `run` must do. Generators are a mechanism which allows for two-way messaging. `run` can take advantage of this by treating every value yielded by the generator as a Promise. Every yielded value will then be attached a `.then` so it can be processed asynchronously. In `then`, the resolved value will be passed back to the generator. Then the process repeats until the iterator has accessed every value that can be retrieved. Also, if the generator ever yields a rejected promise, we will choose to pass the rejected value to the generator as an error (by using `iterator.throw([rejectedValue])`).

We will implement this so that every yielded value is processed asynchronously after the previous. Here is a potential implementation of `run`:

```javascript
function run(generator, ...args) {
  const iterator = generator(args);

  // let "iterObj" be the thing returned by iterator.next() or iterator.throw()
  function handleIterObj(iterObj) {
    try {
      if (iterObj.done) {
        return Promise.resolve(iterObj.value);
      }

      return Promise.resolve(iterObj.value)
        .then(value => handleIterObj(iterator.next(value)))
        .catch(error => handleIterObj(iterator.throw(error)));
    }
    catch(error) {
      return Promise.reject(error);
    }
  }

  return handleIterObj(iterator.next());
}
```

Note that `run` returns a Promise. If the generator passed to `run` returns a value, then that value will be contained in the Promise. Also notice that if an uncaught error occurs in the generator, run will return a Promise which rejects with that error. This is actually behavior which occurs in async functions:

```javascript
async function success() {
  return "success";
}
success().then(console.log);  // "success"

async function failure01() {
  throw new Error("failure");
}
failure01().catch(e => console.log(e.message));  // "failure"

async function failure02() {
  return Promise.reject("failure");
}
failure02().catch(console.log);  // "failure"
```
