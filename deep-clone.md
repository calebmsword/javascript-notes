I recently encountered a situation where it would be convenient to deep copy a JavaScript object. I discovered that this is actually an extremely complex problem. JavaScript provides a native solution through the `structuredClone` method available on the global object. However, it has many limitations that makes it insufficient for some use cases. It is best to discuss all of the potentials pitfalls of a deep-clone algorithm, as well as the design decisions that an algorithm must make. In the process, we will see some of `structuredClone`'s strange design choices and its many limitations.

### deep clone, symbols, and serialization

`structuredClone` is restrictive on what can be cloned because it employs the [structured clone algorithm](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm). The structured clone algorithm is used in many situations where data is serialized to some recipient. An example would be when `postMessage` is called in a Web Worker, which is used to send an object from the Web Worker to the main thread (there is also a function for going the other way around). The point here is that `structuredClone` will only clone objects which can be properly serialized.

Symbols cannot be properly serialized since creating one guarantees a unique value. This leads to the extremely strange fact that `Symbol("foo") === Symbol("foo")` is `false`. So, if you post a message to a Web Worker which contains a symbol and the Web Worker sends a message back which contains that symbol, in general, it is not always possible for the main thread to serialize the symbol into the symbol that was originally sent.

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

You can instantiate new functions like so: `new Function("return 3;")`. But it is not possible to serialize the implementation of a function into a string. This is a good thing, because doing so would introduce massive security holes into JavaScript.

One might hope, then, that the language will introduce a mechanism for securely deep-cloning functions. But it is hard to imagine this ever being the case because of the fact that functions have closures. Deep-cloning a function would require somehow creating a copy of the scope of the original function for the cloned function. I doubt this would ever happen. It might even be impossible.

### deep clone and the prototype chain

`structuredClone` handles the prototype chain strangely. We'll get to that in a second. Before that, let's discuss the merits of NOT cloning the prototype chain. Let the cloned object have the same prototype as the original.

The best use for this is for cloning ES6 classes with methods. We cannot clone a function. But ES6 classes store methods on the prototype of the instance. Since methods dynamically bind `this` to the object which calls them, any function that accesses data with the `this` keyword will refer to data contained in the clone, not the original. 

For the reasons discussed in the previous section, this won't always be a proper clone. The user could easily put a method in the prototype which accesses data from a closure, and the clone would then share this data with the original. This means that if a "clone" shares its prototype with the original, you are trusting that the user attempted to clone a "proper" method which only accesses data through the `this` keyword and arguments passed to the function.

It also possible that you stored non-methods on the prototype chain which are writeable or configurable. In this case, changing the property on one instance will change the property for another. Data will be shared. It is not a proper clone. If you have a use case where you store configurable or writeable non-methods in the prototype chain, then you can easily write an algorithm that walks the prototype chain and clones each object there. For this reason, I think it is reasonable to create a cloning algorithm which does not clone the prototype chain. We can easily use it clone the chain if we need to. A naive example would be

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

This approach won't be perfect. If you encounter a native JavaScript API, for instance, it will have methods. But you can't clone methods. It would probably be better to stop cloning the chain once you reach a prototype with methods on it.

```javascript
const myClone = myCloneAlgorithm(myObject);

function getAllPropertiesOf(object) {
    return [...Object.getOwnPropertyNames(object), ...Object.getOwnPropertySymbols(object)]
}

let tempClone = myClone;
let tempOriginal = myObject;
while (Object.getPrototypeOf(tempOriginal) !== null
       && getAllPropertiesOf(tempOriginal).every(prop =>
              typeof tempOriginal[prop] !== "function")) { 
    const newPrototype = myCloneAlgorithm(Object.getPrototypeOf(tempOriginal));
    Object.setPrototypeOf(tempClone, newPrototype);
    [tempClone, tempOriginal].forEach(temp => {
        temp = Object.getPrototypeOf(temp);
    });
}
```

Now, let's discuss `structuredClone` and its very strange handling of prototypes. If it recognizes that the object to clone has the prototype of a native JavaScript API (`RegExp`, `Date`, `Map`, `Set`, `TypeArray`, etc...), then the cloned object will inherit that prototype. But if the prototype is not that of some recognized JavaScript API, the prototype of the cloned object will be `Object.prototype`.

Sharing the prototype with the original requires trust. But `structuredClone` uses an algorithm which cannot afford trust. It is an algorithm that must guarantee it clones an object that can be safely serialized. If your prototype is that of a native JavaScript API, then `structuredClone` can guarantee that the methods in the prototype access data safely with `this` and arguments passed to the function. An unrecognized prototype might not do the same, so unrecognized prototypes are not inherited in the clone.

I have mixed feelings on `structuredClone`'s refusal to inherit unrecognized prototypes. Perhaps there could have been an second argument which is an object. The object could have a property that, if provided, forces the clone returned by `structuredClone` to inherit the prototype of the original object.

More than anything, I hate that `structuredClone` naively borrows the structured clone algorithm when there is a need for a less strict API for cloning. I wish the function had been named something different and was less strict.

### deep clone and WeakMaps and WeakSets

WeakMaps and WeakSets should not be cloned. WeakMaps and WeakSets contain weak references to the objects given to them, meaning that the object can still get garbage collected in the future. Suppose we tried to clone the data in a WeakMap, and one of its objects is about to get garbage-collected. Should the clone have this data? Should it ignore it? Whatever the answer should be, we cannot know if something will be garbage-collected anyway since the JavaScript specification does not expose this information.

`structuredClone` throws if you try to clone WeakMaps or WeakSets, which is a good choice.

### deep clone and JavaScript APIs

Cloning algorithms, for the most part, walk through all of the properties contained in the object and copy their values. Then, the algorithm sees if any of the properties contain properties and clones those. The process continues until everything is cloned.

However, some data is not accessible through properties. ES6 Maps, for example, have data that can only be accessed through the `get` method. Care must be taken with native JavaScript APIs which access hidden data.

### deep clone and metadata

If your object is sealed or frozen, the "clone" returned by `structuredClone` will not be. In my opinion, this is a mistake, and an update should be made to the ECMAScript specification to fix this.

The property descriptor for any object is also ignored. I believe the reason for this is that property descriptors can have functions (the `get` and `set` accessor properties), and the fact that it is impossible to clone functions made the designers of the structured clone algorithm decide it was best to ignore the property descriptor entirely. I think this a reasonable concern. You could easily create a property with a `get` accessor that returns a value accessed by closure. If the cloned object gets a property with the same `get` accessor, that "cloned" property will access the same memory as the original. The attempt to clone would fail. 

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

It is very easy to create an object which references itself. For example, `const a = {}; a.self = a`. It is not uncommon for nodes in a graph to have children which point back to that node; Nodes in the DOM are just one of many practical examples. Thus, any good cloning algorithm should be able to handle circular references.

Luckily, `structuredClone` does.

### cloning JavaScript native APIs

`structuredClone` ensures that `Date`s, `TypeArray`s, `RegExp`s, `Maps`, and `Sets` are all cloned properly. However, this requires checking the type of the class being cloned.

JavaScript is a dynamic language with monkeypatching. There is nothing to stop a user from monkeypatching native classes. This can cause a well-designed algorithm to suddenly throw errors when a native class is being cloned.

It is also impossible to write a fail-proof method to check the type of an object for the same reason. There are some pretty good solutions (`Object.prototype.toString.call`, or the `instanceof` operator, or how some native API's throw if `this` is bound to an unexpected value), but with enough effort, any of these checks can be circumvented with clever monkeypatching.

A cloning algorithm which attempts to clone native JavaScript APIs is an algorithm that trusts the user to leave native JavaScript APIs alone. Unfortunately, the language does not provide enough mechanisms to protect the programmer from themselves.

If the user doesn't monkeypatch them, `structuredClone` supports every API listed [here](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#supported_types).

### cloning user-created objects

It is impossible for a cloning algorithm to correctly clone all user objects. For example, it is not possible to anticipate an object whose prototype has a method which accesses data through closure instead of the `this` property.

A cloning algorithm should allow for user-injected logic so that users can account for custom types which require specialized logic to clone.

# So, how do we clone JavaScript objects?

It not possible to properly clone every JavaScript object. If your object has functions, `WeakMap`s, or `WeakSet`s, then it cannot be done. If your object has a property whose property descriptor includes accessors, then it also can't be done.

However, if you are cloning an object that 
 - does not have symbol properties or values
 - does not have non-default property descriptors
 - is not sealed or frozen, and no nested values are sealed or frozen
 - has a prototype chain that doesn't need to be cloned
 - has only enumerable properties
 - only uses [these](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#supported_types) native JavaScript APIs
 - uses methods from the prototype chain which only access data through the `this` property
 - is not "deeply" nested, where "deep" is on the order of 1500-2000 layers (or you can guarantee that you code will run on a platform whose `structuredClone` implementation is not recursive)
 - does not own properties which contain functions, `WeakMap`s, or `WeakSet`s (these types can be in the prototype chain, however)

then `structuredClone` will suffice.

If these limitations are too strict for your use case, then [my algorithm](https://github.com/calebmsword/parsec/blob/main/example/example-utils/clone-deep.js) might suffice. The algorithm uses no recursion, supports many of JavaScript's native APIs, retains property descriptors, clones symbols, clones symbol properties and non-enumerable properties, retains the frozen and sealed status of objects, and properly handles circular references. Attempting to copy functions, WeakMaps, or WeakSets causes warning to appear instead of errors. My algorithm also lets you input a custom function that provides custom logic so you can account for complicated custom objects, or even circumvent some design decisions my algorithm makes.

If you use my algorithm, I made some design choices that you must be aware of: 
 - Methods are copied, even though functions cannot be propertly cloned. You can circumvent this with custom logic if necessary.
 - Unsupported types fail with warnings instead of errors. A warning is logged to the console (or an error is sent to a custom logging function that the user provides) and the value is simply cloned as an empty object.
 - The algorithm does not clone any of the types listed [here](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#webapi_types) even though `structuredClone` does.
 - The algorithm will clone property accessors, which can be problematic as discussed previously.
 - The algorithm does not clone the prototype chain. But as shown before, it is easy to use the algorithm to clone the prototype chain of an object if that chain doesn't include methods.

My algorithm is flexible, perhaps too flexible, but I wanted an alternative that was less rigid than `structuredClone`. I wanted the user to clone anything without sending exceptions to the call stack, and I felt this would be okay as long as failures to perform true clones were logged.

# Should we even try to deep clone things?

Probably not.

An appeal of object-oriented programming is encapsulation. This means that objects can have data that we cannot access directly. Cloning something requires accessing all of its data, which completely ignores the principle of encapsulation.

Java might have the right idea here. In Java, classes have the responsiblity of implementing the logic for cloning themselves (`Object::clone` has to be overridden for cloning to be possible). If the object shouldn't be cloned, you can throw an error when the user tries. Perhaps we should do the same in JavaScript.
