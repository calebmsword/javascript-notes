I recently encountered a situation where it would be convenient to deep copy a JavaScript object. I discovered that this is actually an extremely complex problem.

### deep clone and symbols

I originally wanted to use structuredClone since it is a native tool for deep-copying JavaScript objects. However, structuredClone is restrictive on what can be cloned because it employs the algorithm used when calling `postMessage` in a Web Worker. That is, structuredClone will only clone objects which can be properly serialized.

Symbols cannot be properly serialized since creating one guarantees a unique value. This leads to the extremely strange fact that `Symbol("foo") === Symbol("foo")` returns `false`. So, if you post a message to a Web Worker which contains a symbol and the Web Worker sends that symbol back, JavaScript cannot guarantee that the symbol received from the Web Worker will be the same one sent to that Web Worker.

structuredClone will throw an error if you try to clone an object containing a symbol. It is quite unfortunate that there isn't a native solution which removes this restriction, since there are plently of use cases where one would like to clone data that won't be serialized.

### deep clone and functions

structuredClone will also throw if the object contains functions. This is actually a good thing because it is *impossible to deeply clone a function*.

If you google, you will find [hacks](https://stackoverflow.com/a/6772648/22334683) which pretend to clone functions. But they don't actually clone the function. If you create a function which has access to closure:

```javascript
const getIterator = () => {
    let i = 0;
    return () => i++;
}
```

Then the hack shown above, if used on the function returned by `getIterator`, which create a second function which still accesses the `i` variable. So the "cloned" function will still access the same data as the original function. This is a code smell.

You can instantiate new functions like so: `new Function("return 3;");. But it is not possible to serialize the implementation of a function into a string. This is a good thing, because doing so would introduce massive security holes into JavaScript. You don't want a function which uses your AWS access tokens to be easily accessible by a user who opens the console.

### deep clone and the prototype chain

structuredClone does not clone any objects in the prototype chain. The cloned object points to the same prototype as the original object.

This is an acceptable solution. There are some use cases where this is preferable, and other uses cases where it may not be.

The best use for this is for cloning ES6 classes with methods. We cannot clone a function. But ES6 classes store methods on the prototype of the instance. This means that structuredClone won't throw if you clone an ES6 class with methods. Since methods dynamically bind `this` to the object which calls them, this means calling the method on the clone will refer to data contained in the clone, not the original.

However, if you stored non-methods on the prototype chain which are writeable or configurable, then changing the property on one instance will change the property for another. Data will be shared. It is not a proper clone.

If you have a use case where you store data in the prototype chain, then you can easily write an algorithm will walks the prototype chain and clones each object there. For this reason, I think it is a good design decision to create a cloning algorithm which does not clone the prototype chain. We can easily use it to do so if we need to. For example,

```javascript
const myClone = myCloneAlgorithm(myObject);

let tempClone = myClone;
let tempOriginal = myObject;
while (Object.getPrototypeOf(tempOriginal) !== null) { 
    const newPrototype = myCloneAlgorithm(Object.getPrototypeOf(tempOriginal));
    Object.setPrototypeOf(tempClone, newPrototype);
    [tempClone, tempOriginal].forEach(obj => {
        obj = Object.getPrototypeOf(obj);
    });
    
}
```

### deep clone and WeakMaps and WeakSets

WeakMaps and WeakSets should not be cloned. WeakMaps and WeakSets contain weak references to the objects given to them, meaning that the object can still get garbage collected in the future. Suppose we tried to clone the data in a WeakMap, and one of its objects is about to get garbage collected. Should the clone have this data? Should it ignore it? Whatever the answer should be, we cannot know if something is about to be garbage-collected in JavaScript anyway, as the language specification does provide a standard way to expose this information.

structuredClone throws if you try to clone WeakMaps or WeakSets, which is a good choice.

### deep clone and JavaScript APIs

Cloning algorithms, for the most part, walk through all of the properties contained in the object and copy their values. Then, the algorithm sees if any of the properties contain properties and clones those. The process continues until everything is cloned.

However, some data is not accessible through properties. ES6 Maps, for example, have data that can only be accessed through the `get` method. Care must be taken with native JavaScript APIs which access hidden data.

### deep clone and metadata

If your object is sealed or frozen, structuredClone will ignore this in the clone. In my opinion, this is a mistake, and an update should be made to the ECMAScript specification for the structured clone algorithm.

The property descriptor for any object is also ignored. I believe the reason for this is that property descriptors can have functions (the get and set accessor properties), and the fact that it is impossible to clone functions made the designers of the structured clone algorithm decide it was best to ignore the property descriptor entirely. I wish that the algorithm would have preserved enumerability, configurability, and writeability. But oh well.

### deep clone, symbol properties, and non-enumerable properties

Unforunately, structuredClone will not copy any properties on an object that are symbols. I wish this were not the case, but this is likely for serialization concerns.

It is extremely unfortunate that structuredClone *ignores* non-enumerable properties. I can only imagine that this is for the same reasons that structuredClone ignores property descriptors. I still think this is a blunder.

### deep clone and recursion

Unforunately, V8's implementation of structuredClone is recursive. This means that a deeply nested object can blow up the call stack.

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

Spidermonkey does not blow up the call stack. I hope that V8 eventually follows suit.

### cloning JavaScript native APIs

structuredClone ensures that Dates, TypeArrays, RegExps, Maps, and Sets are all cloned properly. However, this requires checking the type of the class being cloned.

JavaScript is a dynamic language with monkeypatching. There is nothing to stop a user from monkeypatching native classes. This can cause a well-designed algorithm to suddenly throw errors when a native class is being cloned.

It is also impossible to write a fail-proof method to check the type of an object for the same reason. There are some pretty good solutions (`Object.prototype.toString.call`, or the `instanceof` operator, or how some native API's throw if `this` is bound to the wrong value), but with enough effort, any of these checks can be circumvented with clever monkeypatching.

A cloning algorithm which attempts to clone native JavaScript APIs is an algorithm that trusts the user to leave native JavaScript APIs alone. Unfortunately, the language does not provide enough mechanisms to protect the programmer from themselves.

structuredClone supports every API listed [here](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#supported_types).

### cloning user-created objects

It is impossible for a cloning algorithm to correctly clone all user objects. For example, it is not possible to anticipate an object whose prototype has a method which accesses data through closure instead of the `this` property.

A cloning algorithm should allow for user-injected logic so that users can account for custom types which require specialized logic to clone.

# So, how do we clone JavaScript objects?

If you are cloning an object that 
 - does not have symbol properties or values
 - does not have non-default property descriptors
 - is not sealed or frozen
 - has a prototype chain that you don't need cloned
 - does not have crucial properties that are non-enumerable
 - only uses [these](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#supported_types) native JavaScript APIs
 - uses custom objects or class instances which access all of their data through the `this` property
 - is not deeply nested (or you can guarantee that you code will run on a platform whose structuredClone implementation is not recursive)
 - does not have functions, weakmaps, or weaksets at the topmost object in the prototype chain

then use `structuredClone`.

If these limitations are too strict, then use my algorithm. My algorithm also lets you input a custom function that provides custom logic so you can account for complicated custom objects, or even circumvent some design decisions my algorithm makes.

My algorithm makes a few specific design choices that you must be aware of. 
 - Methods are copied, even though functions cannot be propertly cloned. You can circumvent this with custom logic if necessary.
 - Unsupported types fail "quietly." A warning is logged to the console (or an error is sent to a custom logging function that the user provides) and the value is simply cloned as an empty object.
 - The algorithm does not clone any of the types listed [here](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#webapi_types), even though structuredClone does.
 - The algorithm does not clone the prototype chain. As shown before, you can use it to do that if you need to.

My algorithm is flexible, perhaps too flexible, but I wanted an alternative that was less rigid than structuredClone. I wanted the user to clone anything without sending exceptions to the call stack, and I felt this would be okay as long as any questionable clone attempts were logged.
