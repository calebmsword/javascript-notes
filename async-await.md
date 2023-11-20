### async-await syntax is just generator function magic

Consider the following:

```javascript
function getAles() {
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

This function is fully equivalent to a generator function, given that we create a utility function which uses the generator function. Let us call the utility function `run`. Before explaining what `run` is or how it is implemented, let us see what the equivalent code would be:

```javascript
function getAles() {
  return fetch("https://api.sampleapis.com/beers/ale");
}

function* showFirstAle() {
  try {
    const res = yield getAles();
    const json = yield res.json();
    console.log(json[0]);
  }
  catch(error) {
    console.log(error);
  }
}

run(showFirstAle).then(console.log);  // "success"
```

The `async` function is now a generator function. All `await`s are replaced with `yield`s. And instead of calling the generator function, we pass it to `run` which magically uses the generator function in some way.

Let us see how to implement `run`:

```javascript
function run(generator, ...args) {
  const iterator = generator(args);

  return Promise.resolve()
    .then(function stepThroughGenerator(prevResolvedValue) {
      const iterObj_ = iterator.next(prevResolvedValue);

      function handleIterObj(iterObj) {
        if (iterObj.done) {
          return iterObj.value;
        }

        return Promise.resolve(iterObj.value)
          .then(stepThroughGenerator)
          .catch(error => {
            iterator.throw(error)
              .then(handleIterObj)
          });
      }

      return handleIterObj(iterObj_);
    });
}
```
