### What ES6 classes are sugar for

You get a 95% accurate understanding of ES6 classes by thinking of them as syntatic sugar for a [constructor function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new#description).
 - static fields are attached to the constructor
   - (non-static methods can still access them with `this.constructor[staticPropertyName]` lmao)
 - A GUESS, NOT CONFIRMED:
   - A "class handler" function is created which returns a constructor function 
 - A GUESS, NOT CONFIRMED:
   - private fields and methods are compiled into variables and functions accessible via closure in the "class handler" function
   - this would explain lots of the weird behavior of private variables (such as how subclasses cannot access a parent's private variables)

```javascript
function createConstructorFunction() {

  // there is likely a compile step when JS parses the class syntax
  // When the parser encounters `this.#privateFunc`, it likely transpiles that into `privateFunc` (no `this`)
  function privateFunc() {
    /* stuff from class */
  }

  // similarly, any `this.#privateVariable` call is likely transpiled into a privateVariable reference (no `this`)
  let privateVariable /* = value from class if provided */;

  // this function would have the name of the class
  const NameOfClass = function(...argsFromConstructor) {
    
    // don't allow people to call constructors without `new`
    if (this === undefined) {
      throw new TypeError(`Class constructor ${NameOfClass.name} cannot be invoked without 'new'`);
    }

    /*
    Everything from the `constructor` method in the class body is (probably) copy-pasted here verbatim
    */
  }

  // a class can extend null; if no extend is specified, then the prototype is Object.prototype
  Class.prototype = Object.create(
    ClassToExtend !== undefined ? ClassToExtend.prototype : Object.prototype);

  // methodsFromClass won't include private methods
  for (const [methodName, method] of Object.entries(methodsFromClass)) {
    NameOfClass.prototype[methodName] = method;
  }

  // staticProps are assigned to the constructor function
  for (const [staticProp, valueOrMethod] of Object.entries(staticsFromClass)) {
    NameOfClass[staticProp] = valueOrMethod;
  }

  // suppose that all code in a static block is transpiled into methods stored in the array `staticBlocks`
  for (const staticBlock of staticBlocks) {
    // make `this` refer to constructor function in all static blocks
    staticBlock.apply(NameOfClass);
  }

  return NameOfClass;
}
```

I should mention that this mental model isn't completely accurate. See, for example, [this](https://stackoverflow.com/a/54861781/22334683) stackexchange post (unfortunately, this was written before private variables were introduced).
