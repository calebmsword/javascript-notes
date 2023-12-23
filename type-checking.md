I am aware of many ways to type-check objects in JavaScript. There are all unreliable, except for one which actually works consistently, but only with custom objects. The other methods, while unreliable, can be used with JavaScript's native objects. So we it is good to understand all of the approaches.

# `instanceof`

The `instanceof` operator takes two arguments. `foo instanceof Bar` crawls through `foo`'s prototype chain and returns true if it contains `Bar.prototype`. An error is thrown if `Bar` is not a function.

This operator is a very loose tool for type-checking. It should be named something like `inheritsfrom` or `subclassof`. It isn't specific about the type of the object.

A stricter check would look something like

```javascript 
function instanceof(object, constructor) {
  return Object.getPrototypeOf(object) === constructor.prototype;
}
```

This better represents what we think of when we see the word `instanceof`. This function is very specific about the type of the object. This still isn't secure, however, because we could easily change an object's prototype before calling this function to create an artifical result. The object could pretend to have been created from a constructor function that it actually wasn't made from.

# `Object.prototype.toString.call`

Try using this function on various objects. The result probably looks something like `"[object Object]"`. If you create an instance of a `Date`, you get `"[object Date]"`.

This is a tempting tool for type-checking native JavaScript classes because the specification requires that many native classes return a specific string. These classes are `Array`, `Function`, `Error`, `Boolean`, `Number`, `String`, `Date`, `RegExp`, `Object`, and the `arguments` object implicitly passed to functions. However, if the object has, anywhere in its prototype chain, the property `Symbol.toStringTag` and its value is a string, the result of `Object.prototype.toString.call` will change. For example:

```javascript
const date = new Date();
console.log(Object.prototype.toString.call(date));  // "[object Date]"
date[Symbol.toStringTag] = "Foo";
console.log(Object.prototype.toString.call(date));  // "[object Foo]"
```

It is very easy to pretend to be a native class when you are not by overriding the `Symbol.toStringTag` property on an object. It is also easy to make a native class pretend not to be a native class. We might think to make an algorithm which can type check `Array`, `Function`, `Error`, `Boolean`, `Number`, `String`, `Date`, `RegExp`, `Object`, or `arguments` by creating a function which deletes the `Symbol.toStringTag` property before calling `Object.prototype.toString.call`. This might look like

```javascript
function isType(object, type) {

    let temps = [];
    let current = object;
    
    // get rid of all Symbol.toStringTag properties in the chain
    while (typeof object[Symbol.toStringTag] === "string") {

        // walk prototypes until we find one that has Symbol.toStringTag property
        while (Object.getPrototypeOf(current) !== null 
               && !current.hasOwnProperty(Symbol.toStringTag))
            current = Object.getPrototypeOf(current);

        // If that Symbol.toStringTag property has a string value, delete it
        if (typeof current[Symbol.toStringTag] === "string") {

            // but remember its value so we can assign it back
            temps.push([current, current[Symbol.toStringTag]]);

            // if we can't delete it, give up
            if (!(delete current[Symbol.toStringTag])) break;
        }
    }

    result = {}.toString.call(object) === `[object ${type}]`;

    temps.forEach(([obj, value]) => {
        try {
            obj[Symbol.toStringTag] = value;
        }
        catch(_) {}
    });
    
    return result;
}
```

This is terrible because it is complex, but not reliable. If the user assigns a non-configurable `Symbol.toStringTag` anywhere in the prototype chain, then this function fails. Instead, we might as well just use `Object.prototype.toString.call(object) === "[object <Type>]"`.

More recent native classes like `Map` and `Set` have a non-writeable `Symbol.toStringTag` in their prototype which forces `Object.prototype.toString.call(object)` to return a specific string. But the property is configurable, so you can still change it by manipulating its property descriptor.


# Calling native methods

Most native methods throw errors if the `this` binding is a function whose prototype is not the prototype. For example:

```javascript
const setSubclass = Object.create(Set.prototype);
try {
  Set.prototype.has.bind(setSubclass)(null));
}
catch(error) {
  console.log("setSubclass does have Set.prototype as its prototype.");
}
```

This is also not secure because we could monkeypatch `Set.prototype` to throw errors when it shouldn't, or to not throw errors at all when it should.

# The One True Type Check

The only secure method I am aware can be performed with factories. For example:

```javascript
// my-object.js

const registry = new WeakSet();

export const getMyObject = () => {
  myObject = Object.freeze({ foo: "bar" });
  registry.add(myObject);
  return myObject;
}

export const isMyObject = candidate => registry.has(candidate);
```

If we do not export the registry from this module, it is safely encapsulated. Then `isMyObject` is a foolproof way to check if an object was created from the `getMyObject` factory.

I have already found myself using this pattern multiple times. It has reached the point where I have created a function which transforms a factory into a type-checkable factory:

```javascript
function getTypeCheckableFactory(name, factory) {
    const registry = WeakSet();
    return {
        [`get${name}`](...args) {
            const result = factory(..args);
            registry.add(result);
            return result;
        },
        [`is${Name}`](candidate) {
            return registry.has(candidate);
        }
    };
}
```

Here is an example of its usage:

```javascript
import getTypeCheckableFactory from "./utils";

const { getWrapper, isWrapper } = getTypeCheckableFactory("Wrapper", () => {
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

const wrapper = getWrapper();
wrapper.set({ foo: "bar" });
console.log(wrapper.get());  // { foo: "bar" }
console.log(isWrapper(wrapper));  // true
```
