I am aware of many ways to type-check objects in JavaScript. There are all unreliable, except for one. However, the reliable method does not work with JavaScript's native classes and it only works with custom objects made in a very specific way. Hence, the unreliable methods still have some use.

Also, keep in mind when I say that a JavaScript object is an instance of the class `ClassName`, I am saying that the object was created with the `ClassName` constructor function using the `new` operator. 

# `instanceof`

The `instanceof` operator takes two arguments like so: `foo instanceof Bar`. `foo` can be any object, and `Bar` must be a suitable constructor function. By default, `instanceof` crawls through `foo`'s prototype chain and returns true if it contains `Bar.prototype`. An error is thrown if `Bar` is not a suitable constructor function.

However, this behavior can be configured. JavaScript looks for the `Symbol.hasInstance` property on the constructor and uses the method associated with it. `Function.prototype`, by default, has a non-configurable, non-writeable `Symbol.hasInstance` that performs the behavior described in the previous paragraph. To add a new value for `Symbol.hasInstance` on a constructor function, you cannot use simple assignment. You have to use `Object.defineProperty`:

```javascript
Bar[Symbol.hasInstance] = newFunction;  // this throws

Object.defineProperty(Bar, Symbol.hasInstance, {
  value: newFunction
});  // this works
```

Any instance of a native JavaScript class can customize the behavior of `instanceof` during runtime meaning that `instanceof` can be unpredictable.

The default behavior of the `instanceof` operator is often not preferable anyway. Usually, given some `object` and a `Constructor`, we want the more specific check `Object.getPrototypeOf(object) === Constructor.prototype` instead of knowing if the prototype is somewhere in the prototype chain. 

# `Object.prototype.toString.call`

Try using this function on various objects. The result probably looks something like `"[object Object]"`. If you create an instance of a `Date`, you get `"[object Date]"`.

This is a popular tool for type-checking native JavaScript classes because the specification requires that `Object.prototype.toString` return specific string values if `this` is bound to any of a particular collection of native JavaScript class instances. These classes are `Array`, `Function`, `Error`, `Boolean`, `Number`, `String`, `Date`, `RegExp`, `Object`, and the class whose instances are the `arguments` objects implicitly passed to functions. Then, for example, we could use `Object.prototype.toString.call(myObject) === "[object Array]"` to test if `myObject` were an array, or `Object.prototype.toString.call(myObject) === "[object Date]"` to check if `myObject` were a date, etc. If we want to check if an object is an `arguments` object, then we do `Object.prototype.toString.call(myObject) === "[object Arguments]"`. 

The default behavior of the `instanceof` operator will not correctly recognize the constructor which made an object if that object has its prototype dynamically changed before the `instanceof` operator is used. However, `Object.prototype.toString.call` does not use the prototype of the object to determine the string output. For example,

```javascript
const date = new Date();
console.log(Object.prototype.toString.call(date));  // "[object Date]"
Object.setPrototypeOf(date, Object.prototype);
console.log(Object.prototype.toString.call(date));  // "[object Date]"
```

It is import to understand that `Object.prototype.toString.call` tells us how the object was constructed. It does not inform us of the object's current prototype. It is possible for it to be different than the `prototype` property of the object's constructor function. 

It seems that `Object.prototype.toString.call` is a reliable method for checking whether an object was constructed from a native JavaScript constructor function. But there is a trivial exploit. The JavaScript specification demands that, if an object has a `Symbol.toStringTag` property whose value is a string anywhere in its prototype chain, then the result of `Object.prototype.toString.call` should change in accordance to that value. For example,

```javascript
const date = new Date();
console.log(Object.prototype.toString.call(date));  // "[object Date]"
date[Symbol.toStringTag] = "Foo";
console.log(Object.prototype.toString.call(date));  // "[object Foo]"
```

We might think to make an algorithm which which deletes any `Symbol.toStringTag` property in the prototype chain before calling `Object.prototype.toString.call`. But this won't work if the user assigns a non-configurable `Symbol.toStringTag` anywhere in the prototype chain. No matter what we do, the result of `Object.prototype.toString.call` can be arbitrarily overridden by an irresponsible `Symbol.toStringTag` property.

More recent native classes like `Map` and `Set` have a non-writeable `Symbol.toStringTag` in their prototype which forces `Object.prototype.toString.call(object)` to return a specific string. But the property is configurable, so you can still change it by manipulating its property descriptor.

Unfortunately, there is an additional exploit that can break this technique: `Object.prototype.toString` can be monkeypatched into any arbitrary function.

# Calling native methods

Most native methods throw errors if the object was not actually created from the constructor for the class. For example:

```javascript
const actualDate = new Date();

const fakeDate = {};
Object.setPrototypeOf(fakeDate, Date.prototype);

Date.prototype.getDay.call(actualDate);
Date.prototype.getDay.call(fakeDate);  // throws
```

`Map`, `Set`, `Array`, and most other native JavaScript classes have similar behavior. Following the previous example, we could create a function like the following to check if an object was a `Map` instance:

```javascript
function isMap(object) {
  try {
    Map.prototype.has.call(object);
    return true;
  }
  catch(error) {
    return false;
  }
}
```

This technique is quite good. Unfortunately, it is also fallible. For example, the user could overwrite `Map.prototype.has` to throw errors when it shouldn't. 

# structuredClone

This is the hackiest hack in this document.

`structuredClone` is a native JavaScript global function which deeply-clones objects. The prototype chain of the object is not cloned. Instead, if the object was created using the constructor function for any of its [supported classes](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#supported_types), the cloned object receives the prototype of the constructor function which created the object.

```javascript
const map = new Map();
const clone = structuredClone(map);
console.log(Object.getPrototypeOf(clone) === Map.prototype));  // true
```

Strangely, this still occurs even if the original object had its prototype changed.

```javascript
const map = new Map();
Object.setPrototypeOf(map, Object.prototype);
const clone = structuredClone(map);
console.log(Object.getPrototypeOf(clone) === Map.prototype));  // true
```

We could create a function like the following which would work for any of the types supported in the structured clone algorithm:

```javascript
function wasConstructedBy(object, constructor) {
  const clone = structuredClone(object);
  return Object.getPrototypeOf(clone) === constructor.prototype;
}
```

This is quite a good technique. However, `structuredClone` can be monkeypatched arbitrarily, so it is not foolproof. `structuredClone` can also be an expensive operation if the cloned object has many properties or is deepy nested.

# The One True Type Check

The only secure method I am aware of can be performed with factories. For example:

```javascript
// my-object.js

const registry = new WeakSet();

const getMyObject = () => {
  myObject = Object.freeze({ foo: "bar" });
  registry.add(myObject);
  return myObject;
}

const isMyObject = candidate => registry.has(candidate);

const myObjectFactory = Object.freeze({
  getMyObject,
  isMyObject
});

export default myObjectFactory;
```

If we do not export the registry from this module, it is safely encapsulated. Then `isMyObject` is a foolproof way to check if an object was created from the `getMyObject` factory.

I have already found myself using this pattern multiple times. It has reached the point where I have created a function which transforms a factory into a type-checkable factory:

```javascript
function getTypeCheckableFactory(factory) {
    const registry = WeakSet();
    return Object.freeze({
        create(...args) {
            const result = factory(..args);
            registry.add(result);
            return result;
        },
        has(candidate) {
            return registry.has(candidate);
        }
    });
}
```

Here is an example of its usage:

```javascript
import getTypeCheckableFactory from "./utils";

const wrapperFactory = getTypeCheckableFactory(() => {
    let value;
    return {
      get() {
          return value;
      }
      set(newValue) {
        value = newValue;
      }
    }
});

const wrapper = wrapperFactory.create();
wrapper.set({ foo: "bar" });
console.log(wrapper.get());  // { foo: "bar" }
console.log(wrapperFactory.has(wrapper));  // true
```

This can be easily used with ES6 classes.

```javascript
import getTypeCheckableFactory from "./utils";
import MyClass from "./my-class";

const MyClassFactory = getTypeCheckableFactory((...args) => new MyClass(...args));
```

The object returned by `getTypeCheckableFactory` is frozen so its methods cannot be changed. If we assign the object to a constant variable, we can trust that the factory object will always have the expected behavior. The technique is foolproof. 

# Summary

- We can use the `instanceof` operator to check if an object has a prototype of a constructor function in its prototype chain. This is not reliable if a `Symbol.hasInstance` property with unexpected behavior is assigned to the constructor function used.
- We can use `Object.prototype.toString.call` to check if an object was created using a native JavaScript constructor function. This is not reliable if the user adds a non-configurable `Symbol.toStringTag` property or, if one is already present, the user changes its value.
- We can get a method from a native prototype, bind it to our object, and then call the bound method. If the method throws, then we know that the object was not created with the constructor for that prototype. This is not reliable if the user monkeypatches the implementation of the prototype.
- We can clone an object with `structuredClone` and check the clone's prototype to see if the object was constructed as one of JavaScript's native classes. But this can be an expensive operation, and the `structuredClone` global function can be arbitrarily monkeypatched.
- For custom types created by an object factory, we get foolproof type checking of custom types if a factory function stores all the objects it creates in an encapsulated registry.  
