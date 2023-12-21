I recently encountered a situation where it would be convenient to deep copy a JavaScript object. I discovered that this is an extremely complex problem, and there are many common situations where it is not possible. JavaScript provides a native solution through the `structuredClone` method available on the global object. However, it has many limitations that makes it insufficient for some use cases. In this document, I was discuss the potential pitfalls of a deep-clone algorithm, as well as the design decisions that an algorithm must make. In the process, we will see some of `structuredClone`'s strange design choices and its many limitations, but we will also see that most of the limitations are actually quite reasonable. I will also share a custom deep clone algorithm I made that has less limitations than that of `structuredClone`.

### deep clone, symbols, and serialization

`structuredClone` is restrictive on what can be cloned because it employs the [structured clone algorithm](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm). You can find the official specification of the structured clone algorithm [here](https://html.spec.whatwg.org/multipage/structured-data.html#safe-passing-of-structured-data). The structured clone algorithm is used in many situations where an object is serialized or deserialized to some recipient. An example would be when `postMessage` is called in a Web Worker, which is used to send an object from the Web Worker to the main thread (there is also a function for going the other way around). Another example is when you use the IndexedDB API to save an object in storage. The point here is that `structuredClone` will only clone objects which can be safely serialized.

Symbols cannot be safely serialized since creating one guarantees a unique value. This leads to the extremely strange fact that `Symbol("foo") === Symbol("foo")` is `false`. This means that if you could save an object with a symbol in the IndexedDB API, close the JavaScript runtime so the symbol registry is emptied, and then start the runtime again and load the object, the returned object will contain a symbol with a different value. This is not a safe serialization.

`structuredClone` will throw an error if you try to clone an object containing a symbol. It would be nice if there were a native solution which removes this restriction. There vast majority of use cases for `structuredClone` won't involve serialization. In those use cases, symbols can be cloned without problem.

### deep clone and functions

`structuredClone` will also throw if it tries to clone an object which contains functions. This is actually a good thing because it is *not always possible to deeply clone a function*.

If you google around, you will find [hacks](https://stackoverflow.com/a/6772648/22334683) which pretend to clone functions. But they don't actually clone the function. They call the same function with extra steps. It is very unfortunate that one popular deep clone library actually uses this hack to "clone" functions.

It is possible to clone *some* functions. It is irresponsible, but I will show you one of the ways.

`Function.prototype.toString`, in most JavaScript runtimes, will return a string containing the exact source code of the function it is called on (this includes whitespace and comments). Note that the specification [demands different behavior](https://tc39.es/ecma262/#sec-function.prototype.tostring) for native JavaScript functions, so you can't clone those. Also note that Safari and Node.js do not fully implement the specification for `Function.prototype.toString`. This is fine because you should not use it anyway.

The simplest way to use this to clone a function is to use `eval`. For example:

```javascript
const myFunc = () => "I am a function";

const clonedMyFunc = eval("() => " + myFunc.toString())();
console.log(clonedMyFunc());  // "I am a function"
```

`const clonedMyFunc = eval(myFunc.toString)` will work if you define `myFunc` using an arrow function, but it will not work if you use the `function` syntax. If you use the `function` syntax, the `eval` will declare a new function in the scope `eval` is called in and return `undefined`. You can handle both cases by creating an anonymous function factory which returns the desired function. The `eval` returns the factory, and by calling the factory, you get a clone of the original function.

This will not always work as expected. The functions declared in `eval` have the scope where the `eval` is executed. If the original function refers to variable names that are not in the scope of the `eval`, chaos will ensue. In strict mode, errors will be thrown. Otherwise, global variables will created, or worse, the function will refer to variables in the scope of the `eval` call.

That is reason enough to never use this approach. But for security concerns, you should absolutely never use `eval` for any reason, ever, especially with custom variables. With the approach I showed, the user can open the JavaScript console on the browser and monkeypatch `Function.prototype.toString` to return a string which defines a function of their choice. **That is, the approach I showed allows users to perform arbitrary code execution**. This is why I said it was irresponsible to show this approach.

Another way to clone functions uses the `Function` constructor with `Function.prototype.toString`. But this is vulnerable to the exact same security concerns as the `eval` approach, so I won't describe it in detail.

**In short, you should never clone functions in JavaScript because there are massive security concerns with the approaches available**. Even if security weren't an issue, the approaches fail for many functions which access values in their closures.

One might hope, then, that the language will introduce a mechanism that will correctly and securely deep-clone a function. But it is hard to imagine this ever being the case because of the fact that functions have closures. Deep-cloning a function would require somehow creating a copy of the scope of the original function for the cloned function. I doubt this would ever happen. It might even be impossible.

I should mention an additional caveat. Some functions can be created from function factories. Much of the time, these functions can be trivially cloned if we have access to the factory. However, it is not possible to know at runtime whether or not a function was created from a factory. There are situations where it could be possible to clone such a function if the user was able to inject custom logic into the algorithm, and we will show an example of this later.

### deep clone and the prototype chain

`structuredClone` handles the prototype chain strangely. We'll get to that in a second. Before that, I will argue that a cloning algorithm should NOT clone the prototype chain. Let the cloned object have the same prototype as the original.

The best use for this is for cloning ES6 classes with methods. ES6 classes store methods on the prototype of the instance. Since methods dynamically bind `this` to the object which calls them, any function that accesses data with the `this` keyword will refer to data contained in the clone, not the original.

For the reasons discussed in the previous section, this won't always be a proper clone. The user could easily put a method in the prototype which accesses data from a closure, and the clone would then share this data with the original. This means that if a "clone" shares its prototype with the original, you are trusting that the user attempted to clone a "proper" method which only accesses data through the `this` keyword and arguments passed to the function.

It also possible that you stored non-methods on the prototype chain which are writeable or configurable. In this case, changing the property on one instance will change the property for another. Data will be shared, and it won't be a proper clone. If you have a use case where you store configurable or writeable non-methods in the prototype chain, then you can easily write an algorithm that walks the prototype chain and clones each object there. For this reason, I think it is reasonable to create a cloning algorithm which does not clone the prototype chain. We can easily use it clone the chain if we need to. A simple example would be

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

This approach won't be perfect. If you encounter a native JavaScript API, for instance, it will have methods. But you can't clone native methods. It might be better to stop once you reach a prototype with methods on it.

```javascript
const myClone = myCloneAlgorithm(myObject);

function getAllPropertiesOf(object) {
    return [...Object.getOwnPropertyNames(object), ...Object.getOwnPropertySymbols(object)]
}

let tempClone = myClone;
let tempOriginal = myObject;
while (Object.getPrototypeOf(tempOriginal) !== null
       && getAllPropertiesOf(tempOriginal).every(prop =>
    const newPrototype = myCloneAlgorithm(Object.getPrototypeOf(tempOriginal));
    Object.setPrototypeOf(tempClone, newPrototype);
    [tempClone, tempOriginal].forEach(temp => {
        temp = Object.getPrototypeOf(temp);
    });
}
```

You could also modify this to copy any link in the chain that doesn't have methods instead of just stopping at the first prototype with methods.

Now, let's discuss `structuredClone` and its strange handling of prototypes. If it recognizes that the object to clone has the prototype of a native JavaScript API (`RegExp`, `Date`, `Map`, `Set`, `TypeArray`, etc...), then the cloned object will inherit that prototype. But if the prototype is not that of some recognized JavaScript API, the prototype of the cloned object will be `Object.prototype`.

Sharing the prototype with the original requires trust. But `structuredClone` uses an algorithm which affords minimul trust. If your prototype is that of a native JavaScript API, then `structuredClone` can guarantee that the methods in the prototype access data safely with `this` and arguments passed to the function. An unrecognized prototype might not do the same, so `structuredClone` errs on the side of caution. Unrecognized prototypes are not inherited in the clone.

I have mixed feelings on `structuredClone`'s refusal to inherit unrecognized prototypes. More than anything, I think it is unfortunate that `structuredClone` naively borrows the structured clone algorithm when there is a need for a less strict API for cloning. I wish the function had been named something different and could optionally be run in a mode that always copies the prototype.

### deep clone and WeakMaps and WeakSets

WeakMaps and WeakSets should not be cloned. WeakMaps and WeakSets contain weak references to the objects given to them, meaning that the object can still get garbage collected in the future. Suppose we tried to clone the data in a WeakMap, and one of its objects is about to get garbage-collected. Should the clone have this data? Should it ignore it? Whatever the answer should be, we cannot know if something will be garbage-collected anyway since the JavaScript specification does not expose this information.

`structuredClone` throws if you try to clone WeakMaps or WeakSets, which is a good choice. It is not actually not possible anyway because WeakMaps and WeakSets do not provide methods for iterating over all of their data.

### deep clone and JavaScript APIs

Cloning algorithms, for the most part, walk through all of the properties contained in the object and copy their values. Then, the algorithm sees if any of the properties contain properties and clones those. The process continues until everything is cloned.

However, some data is not accessible through properties. ES6 Maps, for example, have data that can only be accessed through the `get` method. Care must be taken with native JavaScript APIs which access hidden data to ensure that data is cloned properly.

### deep clone and metadata

If your object is inextensible, sealed, and/or frozen, the "clone" returned by `structuredClone` will not be. In my opinion, this is a mistake, and an update should be made to its specification to fix this.

The property descriptor for any object is also ignored. I believe the reason for this is that property descriptors can have functions (the `get` and `set` accessor properties), and the fact that it is not always possible to clone functions made the designers of the structured clone algorithm decide it was best to ignore the property descriptor entirely. I think there were good intentions with this decision. You could easily create a property with a `get` accessor that returns a value accessed by closure. If the cloned object gets a property with the same `get` accessor, that "cloned" property will access the same memory as the original. The attempt to clone would fail. 

However, the algorithm should have preserved enumerability, configurability, and writeability, and then thrown errors if any property descriptor contained accessors.

### deep clone, symbol properties, and non-enumerable properties

Unforunately, `structuredClone` will not copy any properties on an object that are symbols. The previous discussion on symbols should make it clear why this is case.

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

It is also impossible to write a fail-proof method to check if a type is a native object for the same reason. There are some pretty good solutions (`Object.prototype.toString.call`, or the `instanceof` operator, or how some native API's throw if `this` is bound to an unexpected value), but with enough effort, any of these checks can be circumvented with clever monkeypatching.

It also possible to create objects which falsely indicate they are a native JavaScript class instance, even though they are not. Again, with enough monkeypatching, there isn't a foolproof way to protect from this.

A cloning algorithm which attempts to clone native JavaScript APIs is an algorithm that trusts the user to leave native JavaScript APIs alone. Unfortunately, the language does not provide enough mechanisms to protect the programmer from themselves.

If the user doesn't monkeypatch them, `structuredClone` supports every API listed [here](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#supported_types).

### cloning user-created objects

It is impossible for a cloning algorithm to correctly clone all user-created objects. Let me provide an example of such an object.

```javascript
const registry = new WeakSet();

export const getWrapper = () => {
    let value;
    const wrapper = {
        get() {
            return value;
        }
        set(newValue) {
            value = newValue;
        }
    }
    registry.add(wrapper);
    return wrapper;
}

export const isWrapper = candidate => registry.has(candidate);
```

It is possible to correctly clone the object returned by `getWrapper`, but it requires an API that an algorithm wouldn't be able to anticipate. For example:

```javascript
import { getWrapper, isWrapper } from "./wrapper.js";

function cloneWrapper(wrapper) {
    if (!isWrapper(wrapper)) throw new TypeError("not a wrapper");
    
    const cloned = getWrapper();
    cloned.set(myCloneAlgorithm(wrapper.get()));
    return cloned;
}
```

A cloning algorithm should allow for user-injected logic so that users can account for custom types which require specialized logic to clone.

# So, how do we clone JavaScript objects?

It not possible to properly clone every JavaScript object. If your object has `WeakMap`s or `WeakSet`s, then it cannot be done. If your object has functions, then it might be possible, but there is no secure method for doing so, so we shouldn't clone it regardless. And if your object has a property whose property descriptor includes accessors, then it also might not be possible. Cloning the entire prototype chain is also rarely possible. Most objects inherit from Object.prototype which includes native functions which are impossible to clone.

However, if you are cloning an object that 
 - does not have symbol properties or values
 - does not have non-default property descriptors
 - is not sealed or frozen, and no nested values are sealed or frozen
 - has a prototype chain that points to some native prototype
 - has only enumerable properties
 - only uses [these](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#supported_types) native JavaScript APIs
 - uses methods from the prototype chain which only access data through the `this` property
 - is not "deeply" nested, where "deep" is on the order of 1500-2000 layers (or you can guarantee that you code will run on a platform whose `structuredClone` implementation is not recursive)
 - does not own properties which contain functions, `WeakMap`s, or `WeakSet`s (these types can be in the prototype chain, however)

then `structuredClone` will clone the top-level object in the prototype chain and have the prototype point to a native prototype.

If these limitations are too strict for your use case, then [my algorithm](https://github.com/calebmsword/parsec/blob/main/example/example-utils/clone-deep.js) might suffice. The algorithm uses no recursion, supports many of JavaScript's native APIs, retains property descriptors, clones symbols, clones symbol properties and non-enumerable properties, retains the frozen and sealed status of objects, and properly handles circular references. Attempting to copy functions, WeakMaps, or WeakSets causes warning to appear instead of errors. My algorithm also lets you input a custom function that provides custom logic so you can account for complicated custom objects, or even circumvent some design decisions my algorithm makes.

If you use my algorithm, I made some design choices that you must be aware of: 
 - Methods are copied, even though functions cannot be propertly cloned. You can circumvent this with custom logic if necessary.
 - Unsupported types fail with warnings instead of errors. A warning is logged to the console (or an error is sent to a custom logging function that the user provides) and the value is simply cloned as an empty object.
 - The algorithm does not clone any of the types listed [here](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#webapi_types) even though `structuredClone` does.
 - The algorithm will copy property descriptors which contain property accessors.
 - The algorithm does not clone the prototype chain. But as shown before, it is easy to use the algorithm to clone the prototype chain of an object if that chain doesn't include methods.

My algorithm is flexible, perhaps too flexible, but I wanted an alternative that was less rigid than `structuredClone`. I wanted the user to clone anything without sending exceptions to the call stack, and I felt this would be okay as long as failures to perform true clones were noisily logged.

# Should we even try to deep clone things?

Doing so with a catch-all approach is probably not desirable. There are too many cases where it is simply not possible, or where it is possible but an algorithm wouldn't be able to do it correctly without user input.

Java's approach to deep cloning might be the right one. In Java, classes have the responsiblity of implementing the logic for cloning themselves (`Object::clone` has to be overridden for cloning to be possible). If the object shouldn't be cloned, you can throw an error when the user tries. Perhaps we should do the same in JavaScript.
