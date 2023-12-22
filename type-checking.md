I am aware of four ways to type-check objects in JavaScript. Two of them actually work consistently, but these two don't work with all of JavaScript's native objects. The other two are unreliable, but they can be used with all of JavaScript's native objects. So we should understand all of the approaches.

# `instanceof`

The `instanceof` operator takes two arguments. `foo instanceof Bar` crawls through `foo`'s prototype chain and returns true if it contains `Bar.prototype`. An error is thrown if `Bar` is not a function.

This operator is a very loose tool for type-checking. It should be named something like `inheritsfrom` or `subclassof`. It isn't specific about the type of the object.

A stricter check would look something like

```javascript 
function instanceof(object, constructor) {
  return Object.getPrototypeOf(object) === constructor.prototype;
}
```

This better represents what we think of when we see the word `instanceof`. This function is very specific about the type of the object. This still isn't secure, however, because we could easily change the constructor's prototype before calling this function to create an artifical result.

# `Object.prototype.toString.call`

Try using this function on various objects. The result probably looks something like `"[object Object]"`. If you create an instance of a `Date`, you get `"[object Date]"`.

This is a tempting tool for type-checking native JavaScript classes because the specification requires that many native classes return a specific string. However, if the object has, anywhere in its prototype chain, the property `Symbol.toStringTag` and its value is a string, the result of `Object.prototype.toString.call` will change. For example:

```javascript
const date = new Date();
console.log(Object.prototype.toString.call(date));  // "[object Date]"
date[Symbol.toStringTag] = "Foo";
console.log(Object.prototype.toString.call(date));  // "[object Foo]"
```

It is very easy to pretend to be a native class when you are not by overriding the `Symbol.toStringTag` property on an object. It is also easy to make a native class pretend not to be a native class.

Something stricter would be a function like the following:

```javascript
function isType(object, type) {
    
    let current = object;
    while (Object.getPrototypeOf(current) !== null 
           && !current.hasOwnProperty(Symbol.toStringTag))
        current = Object.getPrototypeOf(current);

    let temp;
    if (typeof current[Symbol.toStringTag] === "string") {
        temp = current[Symbol.toStringTag];
        delete current[Symbol.toStringTag];
    }

    result = {}.toString.call(object) === `[object ${type}]`;

    if (temp !== undefined) current[Symbol.toStringTag] = temp;

    return result;
}
```

This is actually a reliable way to type check for some of JavaScript's native classes. However, some of the newer native types use `Symbol.toStringType` to determine the result of `Object.prototype.toString.call`, so this strict version would only work for native types introduced before ES6.

Checking the ECMAScript specification, we see that this will work for the classes `Array`, `Date`, `Error`, `RegExp`, `Number`, `Boolean`, `String`, and `Function`. It will also work for the `arguments` object that is implicitly passed to functions.

Let us create a type-checker for these types specifically.

```javascript
const nativeTypes = [
  "Arguments",
  "Array",
  "Boolean",
  "Date",
  "Error",
  "Function",
  "Number",
  "RegExp",
  "String"
];

const getTypeChecker = type => object => {
    
    let current = object;
    while (Object.getPrototypeOf(current) !== null 
           && !current.hasOwnProperty(Symbol.toStringTag))
        current = Object.getPrototypeOf(current);

    let temp;
    if (typeof current[Symbol.toStringTag] === "string") {
        temp = current[Symbol.toStringTag];
        delete current[Symbol.toStringTag];
    }

    result = {}.toString.call(object) === `[object ${type}]`;

    if (temp !== undefined) current[Symbol.toStringTag] = temp;

    return result;
}

const is = Object.create(null);

nativeTypes.forEach(type => {
    name = type.substring(0, 1).toLowerCase() + type.substring(1);
    is[name] = getTypeChecker(type);
});

export default is;
```

Then we can use `is` like so:

```javascript
import is from "./type-check";

const arr = [];
console.log(is.array(arr));  // true
arr[Symbol.toStringTag] = "Date";
console.log(Object.prototype.toString.call(arr));  // "[object Date]"
is.array(arr);  // true

const obj = { [Symbol.toStringTag]: "Array" };
console.log(Object.prototype.toString.call(obj));  // "[object Array]"
is.array(obj);  // false
```

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
