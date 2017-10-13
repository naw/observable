#### Credits ####

The idea of explaining Observables as functions first came from Ben Lesh's talk here:

https://www.youtube.com/watch?v=3LKMwkuK0ZE#t=10m47s

I found this approach to be super helpful

## Functions ##

To understand observables, observers, and operators, it is helpful first to think of them as functions.

### What is an observable? ###

Think of an observable as a function that takes a callback and emits a stream of values by calling the callback once for each value. The observable can emit values to the callback synchronously or asynchronously.

This observable emits four values:

```javascript
const helloObservable = (callback) => {
  callback('Hello')
  callback('World')
  callback('How are you')
  setTimeout(() => callback('Today?'), 1000);
}
```

The observable emits values simply by calling the callback once for each value. The callback, in turn, can do whatever it wants with the values.

A simple way to "see" these values being emitted is to specify `console.log` as the callback function:

```javascript
helloObservable(console.log);
```

Effectively, this is the same as this:

```javascript
const helloObservable = () => {
  console.log('Hello')
  console.log('World')
  console.log('How are you')
  setTimeout(() => console.log('Today?'), 1000);
}
```

For convenience if we wanted to see these values directly in the HTML, we could define a `print` callback that prints each value to the DOM, and then call the observable with the `print` callback:

```javascript
const print = (value) => {
  const paragraph = document.createElement('p');
  paragraph.innerHTML = value;
  document.body.appendChild(paragraph);
}

helloObservable(print);
```

<a target="_blank" href="//jsfiddle.net/gladtocode/joyjhLqg">Fiddle</a>

In the above examples, I manually hardcoded strings, the actual values emitted could be anything you want, and could come from any source you want. For example, you could use DOM events as a source and indefinitely stream these to the callback function:

```javascript
const mousePositionObservable = (callback) => {
  document.addEventListener('mousemove', (event) => {
    callback([event.x, event.y]);
  });
}
```

Again, we can easily see what this observable emits by giving it a callback like `print`:

```javascript
mousePositionObservable(print);
```

<a target="_blank" href="//jsfiddle.net/gladtocode/ghcfh2k1">Fiddle</a>

### What is an observer? ###

The `callback` from the above examples is called an `observer`. The `callback` _is_ the `observer`. I only used the terminology `callback` so as not to introduce two new terms at the same time. The observer/callback is merely a function that takes a value and does something with it. Thus far, we have used `console.log` or `print` as our observer, but it could be any function that takes a single value and does something with it (although, of course it doesn't _have_ to actually do anything with it, if it chooses not to).

Therefore, using the observer terminology, the first example would look like this:

```javascript
const integerObservable = (observer) => {
  observer('Hello');
  observer('World');
  observer('How are you');
  setTimeout(() => observer('Today?'), 1000);
}
```

Perhaps, instead of logging each value, we an observer that `alert`'s each value:

```
integerObservable(alert);
```

Or, maybe we want to send each value to an API:

```javascript
(value) => httpLibrary.post("/some_endpoint", { value: value });
```

#### Integer Observable ####

For sake of future examples, I'd like to introduce an observable that emits integers indefinitely:

```javascript
const integerObservable = (observer) => {
  let x = 0;
  setInterval(
    () => {
      x = x + 1;
      observer(x);
    },
    500
  );
};

integerObservable(print);
```

<a target="_blank" href="//jsfiddle.net/gladtocode/8f7epxap">Fiddle</a>

### What is an Operator? ###

An operator is a function that takes an observable and creates/returns a new observable. In other words, it decorates the original observable. It is a higher-order observable.

When you observe the new observable, you don't see the values emitted by the original observable; instead, you see the values emitted by the new observable. These values are usually _based_ on the original values from the original observable, but they don't have to be. The operator itself can return an observable that subscribes to the original observerable to get the original values, and in turn, emits modified values to its own observer.

For example, supposing you had an observable that emitted integers 1,2,3,4,... you could have an operator that creates an observable which gets those integers and in turn emits even numbers 2,4,6,8, ... instead by doubling each value.

```javascript
const doubleOperator = (originalObservable) => {
  const newObservable = function(observer) {
    // observe the original observable, and whenever it emits a value, double it and emit it to our own observer:
    originalObservable((originalValue) => {
      observer(originalValue * 2);
    });
  }
  return newObservable;
}
```

**Remember: The operator is not an observable itself - it is a function that _creates_ an observable based on an existing observable.**

```javascript
const doubleIntegerObservable = doubleOperator(integerObservable);
doubleIntegerObservable(print);
```

<a target="_blank" href="//jsfiddle.net/gladtocode/5x0cnw6z">Fiddle</a>

#### More on operators ####

When an operator decorates an observable, the resulting observable does not _have_ to emit a value for each value that the original observable emits. The operator could be designed so as to throw away values or combine values.

For example, here is an operator that throws away all even integers:

```
const ignoreEvenOperator = (originalObservable) => {
  return (observer) => {
    originalObservable((originalValue) => {
      if (originalValue % 2 == 1) {
        // Only emit the value if it is odd
        observer(originalValue);
      }
    });
  }
}

const oddIntegerObservable = ignoreEvenOperator(integerObservable);
oddIntegerObservable(print);
```

<a target="_blank" href="//jsfiddle.net/gladtocode/mpv4seLw">Fiddle</a>

#### Operators with extra arguments ####

In more advanced cases, an operator can take additional arguments that help determine its behavior. For example, instead of having a `doubleOperator`, we could have a `multiplyOperator` that multiplies by a specified integer rather than multiplying by 2. This integer is simply passed to the operator as the 2nd argument after the original observable that the operator is deacorating:

```javascript
const multiplyOperator = (originalObservable, multiplier) => {
  return function(observer) {
    // Our internalObserver observes the original stream, and for each
    // value emitted, we emit a corresponding value -- the original value
    // multipled by the specified multplier
    const internalObserver = (value) => observer(value * multiplier);
    originalObservable(internalObserver);
  }
}

const fivesObservable = multiplyOperator(integerObservable, 5);
fivesObservable(print);
```

<a target="_blank" href="//jsfiddle.net/gladtocode/t7v76my4">Fiddle</a>

#### Operator composition ####

Since operators take an observable and return an observable, we can easily apply more than one operator (decorator) to an observable. For example, we could start with the integers, throw out even numbers, and then multiply by 10, resulting in 10,30,50,70,90,....

```javascript
const oddTensObservable = multiplyOperator(ignoreEvenOperator(integerObservable), 10);
oddTensObservable(print);
```

<a target="_blank" href="//jsfiddle.net/gladtocode/fcr8047y">Fiddle</a>

## Objects

Suppose observers were objects instead of functions. By wrapping the observable function within an object, we are able to achieve a chainable dot syntax like this:

```javascript
const oddTensObservable = integerObservable.ignoreEven().multiply(10);
oddTensObservable.subscribe(print);
```

### How does this work? ###

First, imagine we have an `Observable` class:

```javascript
class Observable {
  constructor(observableFunction) {
    this.observableFunction = observableFunction;
    this.subscribe = this.subscribe.bind(this):
  }

  subscribe(observer) {
    return this.observableFunction(observer);
  }
}
```

We can take an observable function and turn it into an instance of the Observable class:

```javascript
const observable = new Observable(integerObservable);
```

Notice that we can "observe" this observable object by calling its `subscribe` method:

```javascript
observable.subscribe(print);
```

Fiddle: <a target="_blank" href="//jsfiddle.net/gladtocode/jxztvh02">Fiddle</a>

Notice: This is just a thin object wrapper around the old functions.

### Chaining ##

Next, imagine that we our operators are actually methods on the Observable class which apply the operator to the underlying observable function and return a brand new instance of the Observable class constructed using the decorated observable function instead of the original observable function. This means we can chain these operators directly onto observables and obtain the final decorated observable:


```javascript
class Observable {
  constructor(observableFunction) {
    this.observableFunction = observableFunction;
    this.subscribe = this.subscribe.bind(this);
  }

  subscribe(observer) {
    return this.observableFunction(observer);
  }

  multiply(multiplier) {
    return new Observable(multiplyOperator(this.observableFunction, multiplier));
  }

  ignoreEven() {
    return new Observable(ignoreEvenOperator(this.observableFunction));
  }
}
```

Now try it:

```javascript
const observable = new Observable(integerObservable);
observable.ignoreEven().multiply(10).subscribe(print);
```

<a target="_blank" href="//jsfiddle.net/gladtocode/zqezr7ww">Fiddle</a>


## RxJS ##

This object-based Observable pattern is the essence of how RxJS works. You can create an Observable object and then chain various operators on it to yield a final observable that you want to observe. RxJS provides dozens of operators to do fancy things like mapping, debouncing, filtering, etc.

Additionally, RxJS provides functions for creating Observables out of other kinds of objects (like functions, promises, event bindings, etc.)
