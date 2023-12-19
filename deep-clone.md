I recently encountered a situation where it would be convenient to deep copy a JavaScript object. I discovered that this is actually an extremely complex problem. JavaScript provides a native solution through the `structuredClone` method available on the global object. However, it has many limitations that make insufficient for some use cases. It is best to discuss all of the pitfalls and design decisions that a deep-clone algorithm must make. In the process, we will see some of `structuredClone`'s strange design choices and its many limitations. 

### deep clone, symbols, and serialization

`structuredClone` is restrictive on what can be cloned because it employs the [structured clone algorithm](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm), which is the algorithm used when calling `postMessage` in a Web Worker. `postMessage` is used to send an object from the main thread to a Web Worker (or the other way around). This means that `structuredClone` will only clone objects which can be properly serialized.

Symbols cannot be properly serialized since creating one guarantees a unique value. This leads to the extremely strange fact that `Symbol("foo") === Symbol("foo")` returns `false`. So, if you post a message to a Web Worker which contains a symbol and the Web Worker sends a message back which contains that symbol, it is not possible for the main thread to serialize the symbol into the symbol was originally sent for every use case.

`structuredClone` will throw an error if you try to clone an object containing a symbol. It is quite unfortunate that there isn't a native solution which removes this restriction, since there are plently of use cases where one would like to clone data that won't be serialized.

### deep clone and functions

`structuredClone` will also throw if it tries to clone an object which contains functions. This is actually a good thing because it is *impossible to deeply clone a function*.

If you google around, you will find [hacks](https://stackoverflow.com/a/6772648/22334683) which pretend to clone functions. But they don't actually clone the function. If you create a function which has access to closure:

```javascript
const getIterator = () => {
    let i = 0;
    return () => i++;
}
```

then the hack shown above, if used on the function returned by `getIterator`, which create a second function which still accesses the `i` variable. So the "cloned" function will still access the same data as the original function. This is a code smell.

You can instantiate new functions like so: `new Function("return 3;")`. But it is not possible to serialize the implementation of a function into a string. This is a good thing, because doing so would introduce massive security holes into JavaScript. You don't want a function which uses your AWS access tokens to be easily accessible by a user who opens the console.

### deep clone and the prototype chain

`structuredClone` does not clone any objects in the prototype chain. The cloned object points to the same prototype as the original object.

This is an acceptable solution. There are some use cases where this is preferable, and other uses cases where it may not be.

The best use for this is for cloning ES6 classes with methods. We cannot clone a function. But ES6 classes store methods on the prototype of the instance. This means that `structuredClone` won't throw if you clone an instance of an ES6 class which has methods. Since methods dynamically bind `this` to the object which calls them, any function that accesses data with the `this` keyword will refer to data contained in the clone, not the original. For the reasons discussed in the previous section, this won't always be the case, but it often is.

However, if you stored non-methods on the prototype chain which are writeable or configurable, then changing the property on one instance will change the property for another. Data will be shared. It is not a proper clone.

If you have a use case where you store data in the prototype chain, then you can easily write an algorithm will walks the prototype chain and clones each object there. For this reason, I think it is reasonable to create a cloning algorithm which does not clone the prototype chain. We can easily use it clone the chain if we need to. For example,

```javascript
const myClone = myCloneAlgorithm(myObject);

let tempClone = myClone;
let tempOriginal = myObject;
while (Object.getPrototypeOf(tempOriginal) !== null) { 
    const newPrototype = myCloneAlgorithm(Object.getPrototypeOf(tempOriginal));
    Object.setPrototypeOf(tempClone, newPrototype);
    [tempClone, tempOriginal].forEach(temp => {
        temp = Object.getPrototypeOf(temp);
    });
}
```

### deep clone and WeakMaps and WeakSets

WeakMaps and WeakSets should not be cloned. WeakMaps and WeakSets contain weak references to the objects given to them, meaning that the object can still get garbage collected in the future. Suppose we tried to clone the data in a WeakMap, and one of its objects is about to get garbage collected. Should the clone have this data? Should it ignore it? Whatever the answer should be, we cannot know if something will be garbage-collected anyway since the JavaScript specification does not expose this information.

`structuredClone` throws if you try to clone WeakMaps or WeakSets, which is a good choice.

### deep clone and JavaScript APIs

Cloning algorithms, for the most part, walk through all of the properties contained in the object and copy their values. Then, the algorithm sees if any of the properties contain properties and clones those. The process continues until everything is cloned.

However, some data is not accessible through properties. ES6 Maps, for example, have data that can only be accessed through the `get` method. Care must be taken with native JavaScript APIs which access hidden data.

### deep clone and metadata

If your object is sealed or frozen, the "clone" returned by `structuredClone` will not be. In my opinion, this is a mistake, and an update should be made to the ECMAScript specification to fix this.

The property descriptor for any object is also ignored. I believe the reason for this is that property descriptors can have functions (the `get` and `set` accessor properties), and the fact that it is impossible to clone functions made the designers of the structured clone algorithm decide it was best to ignore the property descriptor entirely. I think this a reasonable concern. You could easily create a property with a `get` accessor that returns a value accessed by closure. If the cloned object gets a property with the same `get` accessor, that "cloned" property  the same memory as the original. The attempt to clone would fail. 

However, the algorithm should have preserved enumerability, configurability, and writeability, and then thrown errors if any property descriptor contained accessors.

### deep clone, symbol properties, and non-enumerable properties

Unforunately, `structuredClone` will not copy any properties on an object that are symbols. I wish this were not the case, but this is likely for serialization concerns.

It is extremely unfortunate that `structuredClone` *ignores* non-enumerable properties. I think this is a blunder.

### deep clone and recursion

Unforunately, V8's implementation of `structuredClone` is recursive. This means that a deeply nested object can blow up the call stack.

```javascript
const obj = {};
let next = obj;
for (let i = 0; i < 10000; i++) {
    next.b = {};
    next = next.b;
}

// This blows up the call stack on Chrome
const clonedObj = structuredClone(obj);
```

Spidermonkey's implementation of `structuredClone` does not blow up the call stack. I hope that V8 eventually follows suit.

### deep clone and circular references

Before `structuredClone` was introduced, the most common hack for deep cloning objects was to use `const myClone = JSON.parse(JSON.stringify(myObject))`. This would only work for objects with primitives and/or arrays of primitives. Many JavaScript APIs like `Date` or `RegExp` would be lost in the process. And the hack throws an error if the object had circular references.

It is very easy to create an object which references itself. For example, `const a = {}; a.self = a`. It is not uncommon for graph data structures to have children which point back to the graph. Thus, any good cloning algorithm should be able to handle circular references.

Luckily, `structuredClone` does.

### cloning JavaScript native APIs

`structuredClone` ensures that `Date`s, `TypeArray`s, `RegExp`s, `Maps`, and `Sets` are all cloned properly. However, this requires checking the type of the class being cloned.

JavaScript is a dynamic language with monkeypatching. There is nothing to stop a user from monkeypatching native classes. This can cause a well-designed algorithm to suddenly throw errors when a native class is being cloned.

It is also impossible to write a fail-proof method to check the type of an object for the same reason. There are some pretty good solutions (`Object.prototype.toString.call`, or the `instanceof` operator, or how some native API's throw if `this` is bound to the wrong value), but with enough effort, any of these checks can be circumvented with clever monkeypatching.

A cloning algorithm which attempts to clone native JavaScript APIs is an algorithm that trusts the user to leave native JavaScript APIs alone. Unfortunately, the language does not provide enough mechanisms to protect the programmer from themselves.

If the user doesn't monkeypatch them, `structuredClone` supports every API listed [here](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#supported_types).

### cloning user-created objects

It is impossible for a cloning algorithm to correctly clone all user objects. For example, it is not possible to anticipate an object whose prototype has a method which accesses data through closure instead of the `this` property.

A cloning algorithm should allow for user-injected logic so that users can account for custom types which require specialized logic to clone.

# So, how do we clone JavaScript objects?

If your object has functions, `WeakMap`s, or `WeakSet`s, then it cannot be done. 

However, if you are cloning an object that 
 - does not have symbol properties or values
 - does not have non-default property descriptors
 - is not sealed or frozen, or no nested values are sealed or frozen
 - has a prototype chain that doesn't need to be cloned
 - has only enumerable properties
 - only uses [these](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#supported_types) native JavaScript APIs
 - uses methods from the prototype chain which only access data through the `this` property
 - is not "deeply" nested, where "deep" is on the order of 1500-2000 layers (or you can guarantee that you code will run on a platform whose `structuredClone` implementation is not recursive)
 - does not own properties which contain functions, `WeakMap`s, or `WeakSet`s (these types can be in the prototype chain, however)

then use `structuredClone`.

If these limitations are too strict for your use case, then [my algorithm](https://github.com/calebmsword/parsec/blob/main/example/example-utils/clone-deep.js) might suffice. The algorithm uses no recursion, supports many of JavaScript's native APIs, retains property descriptors, clones symbols, clones symbol properties and non-enumerable properties, retains the frozen and sealed status of objects, and properly handles circular references. Attempting to copy functions, WeakMaps, or WeakSets causes warning to appear instead of errors. My algorithm also lets you input a custom function that provides custom logic so you can account for complicated custom objects, or even circumvent some design decisions my algorithm makes.

If you use my algorithm, I made some design choices that you must be aware of: 
 - Methods are copied, even though functions cannot be propertly cloned. You can circumvent this with custom logic if necessary.
 - Unsupported types fail with warnings instead of errors. A warning is logged to the console (or an error is sent to a custom logging function that the user provides) and the value is simply cloned as an empty object.
 - The algorithm does not clone any of the types listed [here](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#webapi_types) even though `structuredClone` does.
 - The algorithm does not clone the prototype chain. But as shown before, it is easy to use the algorithm to clone the prototype of a clone.

My algorithm is flexible, perhaps too flexible, but I wanted an alternative that was less rigid than `structuredClone`. I wanted the user to clone anything without sending exceptions to the call stack, and I felt this would be okay as long as failures to perform true clones were logged.