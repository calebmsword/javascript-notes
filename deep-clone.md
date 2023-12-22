I recently wanted to deep copy a JavaScript object. In the process, I discovered that this is a complex and interesting problem. JavaScript provides a native solution through the `structuredClone` method available on the global object, but it has many limitations. In this document, I was discuss the potential pitfalls of a deep-clone algorithm as well as the design decisions that such an algorithm must make. In the process, we will see `structuredClone`'s strange design choices and my opinions on them. I will also share a custom deep clone algorithm I made that has less limitations than that of `structuredClone`.

### deep clone, symbols, and serialization

`structuredClone` is restrictive on what can be cloned because it employs the [structured clone algorithm](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm). You can find the official specification of the structured clone algorithm [here](https://html.spec.whatwg.org/multipage/structured-data.html#safe-passing-of-structured-data). The structured clone algorithm is used in many situations where an object is serialized or deserialized to some recipient. An example would be when `postMessage` is called in a Web Worker, which is used to send an object from the Web Worker to the main thread. Another example is when you use the IndexedDB API to save an object in storage. In short, the structured clone algorithm will only clone objects which can be safely serialized.

Symbols cannot be safely serialized since creating one guarantees a unique value. For example, `Symbol("foo") === Symbol("foo")` is `false`. This means that if you save an object with a symbol in the IndexedDB API, close the JavaScript runtime so the symbol registry is emptied, and then start the runtime again and load the object, the returned object will contain a symbol with a different value. This is not a safe serialization.

`structuredClone` will throw an error if you try to clone an object containing a symbol. It would be nice if there were a native solution which removes this restriction, as most cases for `structuredClone` won't involve serialization.

Immediately, we see the design flaw of `structuredClone`. It is not a function which deeply clones objects. It is solves a more specific problem: it deep clones objects **if they can be safely serialized**. The reason it was built this way was because it was easy. The structured clone algorithm specification, by coincidence, happened to deep clone serializable objects. Many JavaScript runtimes implemented a function to do this. It was realized that, since there is a desire for deep cloning, it would be nice for the JavaScript community if this "deep clone function" was exposed.

The problem is that they stopped there. It would have been nice if the committee which introduced `structuredClone` took the time to extend the functionality to solve a wider collection of use cases. Instead, we are given a function that cannot clone one of the basic primitive types in JavaScript.

### deep clone and functions

`structuredClone` will also throw if it tries to clone an object which contains functions. This is a reasonable design decision because it is *not always possible to deeply clone a function*.

If you google around, you will find [hacks](https://stackoverflow.com/a/6772648/22334683) which pretend to clone functions. But they don't really clone the function. A new `Function` object is created, but all it does is call the original function. This means that the closure of the original function is shared with the new function. The "cloned" function and the original function can share data if the original function accesses its closure, so it is not a true clone. It is very unfortunate that one popular deep clone library actually uses this hack to "clone" functions.

There are forbidden techniques that use `Function.prototype.toString` to "clone" a function. But like the previous hack, they also struggle with closures. it would be irresponsible of me to describe these techniques in detail because they are rife with security issues, so I won't.

One might imagine, then, that the specification committee could introduce a mechanism that will correctly and securely deep-clone a function. But it is hard to imagine this ever being the case because of closure. Deep-cloning a function would require somehow creating a copy of the closure of the original function for the cloned function. I can't imagine this would be a good idea.

There is one situation where a function can be reliably cloned, and that is if the function is created from a well-designed factory. Here is a simple example:

```javascript
function getRepeater() {
    return value => console.log(value);
}
```

This factory creates a function which simply logs whatever value is provided to it. This function does not access any data outside of the scope of the factory. If we want a copy of this function, we simply have to call the factory again.

The issue is determining whether or not the function came from the factory. The solution is create a module which exports the factory and keeps track of all of the functions which have been made:

```javascript
// repeater.js

const registry = Set();

export function getRepeater() {
    const repeater = value => console.log(value);
    registry.add(repeater);
    return repeater;
}

export function isRepeater(candidate) {
    return registry.has(candidate);
}
```

As long as we encapsulate the registry in the module, then we have a foolproof method for checking if the function was returned from the factory using `isRepeater`. That means that, any time we try to clone a function, we could check if it is a repeater. If it is, we can call the factory to create a proper clone.

Here is an example of a factory function whose spawn cannot be cloned.

```javascript
let i = 0;
function getIterator() {
    return () => i++;
}
```

This functions returned by `getIterator` has closure which accesses data outside of the scope of the factory function. Any function created by the iterator will share data. We can make this function cloneable with the following change:

```javascript
function getIterator() {
    let i = 0;
    return () => i++;
}
```

The functions returned by `getIterator` still access data in their closure, but that closure in contained within the factory. Every call of `getIterator` creates a new `i` variable, so every iterator accesses different data. If we use the "registry" technique showed in the repeater example, then iterators could also be cloned.

Unfortunately, there is no way to tell `structuredClone` about function factories, so well-designed factory functions cannot have their spawn be cloned with `structuredClone`. We should have an algorithm would allow for user-injected logic so that we could correctly clone functions from well-designed factories.

### deep clone and the prototype chain

I believe that a cloning algorithm should NOT clone the prototype chain. Instead, let the cloned object share the same prototype as the original.

The best use for this is for cloning ES6 classes with methods. ES6 classes store methods on the prototype of the instance. Since methods dynamically bind `this` to the object which calls them, any function that accesses data with the `this` keyword will refer to data contained in the clone, not the original.

For the reasons discussed in the previous section, this won't always be a proper clone. The user could easily put a method in the prototype which accesses data from a closure, and the clone would then share this data with the original. This means that if a "clone" shares its prototype with the original, you are trusting that the user attempted to clone a "proper" method which only accesses data through the `this` keyword and arguments passed to the function.

It also possible that you stored non-methods on the prototype chain which are writeable or configurable. In this case, changing the property on one instance will change the property for another. Data will be shared, and it won't be a proper clone. If you have a use case where you store configurable or writeable non-methods in the prototype chain, then you can easily write an algorithm that walks the prototype chain and clones each object there. For this reason, I think it is reasonable to create a cloning algorithm which does not clone the prototype chain. We can easily use it clone the chain if we need to. A simple example would be

```javascript
// myClone now has the prototype of myObject
const myClone = myCloneAlgorithm(myObject);

let tempClone = myClone;
let tempOriginal = myObject;

// We can't clone native JavaScript methods, so stop if we hit Object.prototype
while (![null, Object.prototype].includes(Object.getPrototypeOf(tempOriginal))) { 
    const newPrototype = myCloneAlgorithm(Object.getPrototypeOf(tempOriginal));
    Object.setPrototypeOf(tempClone, newPrototype);
    [tempClone, tempOriginal].forEach(temp => {
        temp = Object.getPrototypeOf(temp);
    });
}
```

This approach won't be perfect. If you encounter a native JavaScript API, for instance, it will have methods. But you can't clone native methods. It might be better to stop once you reach any prototype with methods on it.

```javascript
const myClone = myCloneAlgorithm(myObject);

// This will get non-enumerable properties
function getAllPropertiesOf(object) {
    return [...Object.getOwnPropertyNames(object), ...Object.getOwnPropertySymbols(object)]
}

let tempClone = myClone;
let tempOriginal = myObject;
while (Object.getPrototypeOf(tempOriginal) !== null
       && getAllPropertiesOf(tempOriginal).every(prop =>
           tempOriginal[prop] !== "function")) {
    const newPrototype = myCloneAlgorithm(Object.getPrototypeOf(tempOriginal));
    Object.setPrototypeOf(tempClone, newPrototype);
    [tempClone, tempOriginal].forEach(temp => {
        temp = Object.getPrototypeOf(temp);
    });
}
```

You could also modify this to copy any link in the chain that doesn't have methods instead of just stopping at the first prototype with methods.

Now, let's discuss `structuredClone` and its strange handling of prototypes. If it recognizes that the object to clone has the prototype of a native JavaScript API (`RegExp`, `Date`, `Map`, `Set`, `TypeArray`, etc...), then the cloned object will inherit that prototype. But if the prototype is not that of some recognized JavaScript API, the prototype of the cloned object will be `Object.prototype`.

Sharing the prototype with the original requires trust. But `structuredClone` uses an algorithm which affords minimul trust. Native JavaScript classes have methods which only access data through `this` and arguments passed to the function. If your prototype is that of a native JavaScript API, then `structuredClone` can guarantee that the methods in the prototype access data safely. An unrecognized prototype might not do the same, so `structuredClone` errs on the side of caution. Unrecognized prototypes are not inherited in the clone.

I have mixed feelings on `structuredClone`'s refusal to inherit unrecognized prototypes. I wish the function had been named something different and could optionally be run in a mode that always copies the prototype.

### deep clone and WeakMaps and WeakSets

WeakMaps and WeakSets should not be cloned. WeakMaps and WeakSets contain weak references to the objects given to them, meaning that the object can still get garbage collected in the future. Suppose we tried to clone the data in a WeakMap, and one of its objects is about to get garbage-collected. Should the clone have this data? Should it ignore it? Whatever the answer should be, we cannot know if something will be garbage-collected anyway since the JavaScript specification does not expose this information.

`structuredClone` throws if you try to clone WeakMaps or WeakSets, which is one of the few restrictions of `structuredClone` this is also correct for the general use case. It is not even possible to clone WeakMaps or WeakSets anyway because WeakMaps and WeakSets do not provide methods for iterating over all of their data.

### deep clone and JavaScript APIs

Cloning algorithms, for the most part, walk through all of the properties contained in the object and copy their values. Then the algorithm sees if any of the properties contain properties and clones those. The process continues until everything is cloned.

However, some data is not accessible through properties. ES6 Maps, for example, have data that can only be accessed through the `get` method. Care must be taken with native JavaScript APIs which access hidden data to ensure that data is cloned properly.

### deep clone and metadata

If your object is inextensible, sealed, and/or frozen, the "clone" returned by `structuredClone` will not be. Perhaps the designers of the structured clone algorithm believed it didn't make sense for serializable data to be immutable and that the recipient should be free to do whatever they like with it. Whatever the reason for this design choice, I wish that the native deep-clone function in JavaScript didn't share this behavior.

The property descriptor for any object is also ignored. I believe the reason for this is that property descriptors can have functions (the `get` and `set` accessor properties), and since it is not possible to clone every function, the designers of the structured clone algorithm decide it was best to ignore the property descriptor entirely. I think there were good intentions with this decision. You could easily create a property with a `get` accessor that returns a value accessed by closure. If the cloned object gets a property with the same `get` accessor, that "cloned" property will access the same memory as the original. The attempt to clone would fail. 

However, the native deep-clone algorithm should have preserved enumerability, configurability, and writeability. It would be acceptable if it threw errors if any property descriptor contained accessors, but I also wouldn't mind if it trusted the user to clone objects with safe acessors.

### deep clone, symbol properties, and non-enumerable properties

Unforunately, `structuredClone` will not copy any properties on an object that are symbols. The previous discussion on symbols should make it clear why this is case.

It also ends up that `structuredClone` *ignores* non-enumerable properties. It is extremely unfortunate that the only native deep-copy method JavaScript forces this restriction.

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

It is very easy to create an object which references itself. It is also practical to do so. It is not uncommon for nodes in a graph to have children which point back to that node; `Node`s in the DOM are just one of many practical examples. Therefore, any good cloning algorithm should be able to handle circular references. `structuredClone` does.

### cloning JavaScript native APIs

`structuredClone` ensures that `Date`s, `TypeArray`s, `RegExp`s, `Maps`, and `Sets` are all cloned properly. However, this requires checking the type of the class being cloned.

JavaScript is a dynamic language with monkeypatching. There is nothing to stop a user from monkeypatching native classes. This can cause a well-designed algorithm to suddenly throw errors when a native class is being cloned. It is also impossible to write a fail-proof method to check if a type is a native object for the same reason. There are some pretty good solutions (`Object.prototype.toString.call`, or the `instanceof` operator, or how some native API's throw if `this` is bound to an unexpected value), but with enough effort, any of these checks can be circumvented with clever monkeypatching. It also possible to create objects which falsely indicate they are a native JavaScript class instance, and there isn't a foolproof way to protect from this.

A cloning algorithm which attempts to clone native JavaScript APIs is an algorithm that trusts the user to leave native JavaScript APIs alone. If the user doesn't monkeypatch them, `structuredClone` supports every API listed [here](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#supported_types). This is one of the few situations where `structuredClone` affords trust to the user.

### cloning user-created objects

Like with functions, objects can also be cloneable if they are created from well-designed factories. Here is a simple factory which creates a object which wraps an encapsulated value.

```javascript
// wrapper.js

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

The object created by this factory does not access any data outside the scope fo the factory function. The API of the object allows us to manipulate the value it encapsulates. We also provided a reliable method for determining whether an object was created by the factory. This means that we can properly clone the objects from this factory if it encapsulates a cloneable value. Here is one way we could do it:

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

It not possible to properly clone every JavaScript object. If your object has `WeakMap`s or `WeakSet`s, then it cannot be done. If your object has functions, it often impossible. If your object has a property whose property descriptor includes accessors, then it may also be impossible. Cloning the entire prototype chain is also rarely possible. Most objects inherit from `Object.prototype` which includes native functions which are impossible to clone.

However, if you are willing to allow a "cloned" object have a native JavaScript prototype as its prototype, then you can use `structuredClone`. But only if the object you wish to clone
 - does not have symbol properties or values
 - does not have non-default property descriptors
 - is not sealed or frozen, and no nested values are sealed or frozen
 - has a prototype chain that points to some native prototype
 - has only enumerable properties
 - only uses [these](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#supported_types) native JavaScript APIs
 - uses methods from the prototype chain which only access data through the `this` property
 - is not "deeply" nested, where "deep" is on the order of 1500-2000 layers (or you can guarantee that you code will run on a platform whose `structuredClone` implementation is not recursive)
 - does not own properties which contain functions, `WeakMap`s, or `WeakSet`s

then `structuredClone` will clone the top-level object in the prototype chain and have the prototype point to a native prototype.

If these limitations are too strict for your use case, then [my algorithm](https://github.com/calebmsword/parsec/blob/main/example/example-utils/clone-deep.js) might suffice. The algorithm uses no recursion, supports many of JavaScript's native APIs, retains property descriptors, clones symbols, clones symbol properties and non-enumerable properties, retains the extensible, frozen and sealed status of objects, and properly handles circular references. Attempting to copy functions, WeakMaps, or WeakSets causes warning to appear instead of errors. My algorithm also lets you input a custom function that provides custom logic so you can account for complicated custom objects, or even circumvent some design decisions my algorithm makes.

If you use my algorithm, I made some design choices that you must be aware of: 
 - Methods are copied, even though functions cannot be propertly cloned. You can circumvent this with custom logic if necessary.
 - Unsupported types fail with warnings instead of errors. A warning is logged to the console (or an error is sent to a custom logging function that the user provides) and the value is simply cloned as an empty object.
 - The algorithm does not clone any of the types listed [here](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#webapi_types) even though `structuredClone` does.
 - The algorithm will copy property descriptors which contain property accessors.
 - The algorithm does not clone the prototype chain. But as shown before, it is easy to use the algorithm to clone the prototype chain of an object if that chain doesn't include methods.

My algorithm is flexible, perhaps too flexible, but I wanted an alternative that was less restrictive than `structuredClone`. I wanted the user to clone anything without sending exceptions to the call stack, and I felt this would be okay as long as failures to perform true clones were noisily logged.
