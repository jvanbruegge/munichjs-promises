# Promise considered harmful

Jan van Brügge

-------------------------------

## Javascript has many broken parts

-------------------------------

## most of them are not really used

You may saw them the last time presentation
(Javascript - The bad parts)

-------------------------------

## But there is an ubiquitous feature that is inheritly broken

-------------------------------

#

(Extra slide for dramatic effect - but you can probably already guess from the title)

# THE PROMISE

-------------------------------

![](https://pbs.twimg.com/profile_images/2727748211/c3d0981ae770f926eedf4eda7505b006.jpeg)

-------------------------------

# Are you nuts?

![Callback hell](https://www.altexsoft.com/media/2017/10/6cfYoU3.png)

-------------------------------

## Problem 1: `then()` is a mix of `map()` and `flatMap()`

Automaticly flattens if Promise is returned.

>- No control!
>- Many data structures are mappable (ie array), but no common interface
>- Also tries to call `then()` on objects you return!

-------------------------------

## Problem 2: Never synchronous

-------------------------------

### What do these programs log?

```js
const callback = x => console.log(x);

console.log('before');
Promise.resolve(42).then(callback);
console.log('after');
```

<div class="fragment fade-in">
```
before
after
42
```
</div>

-------------------------------

### What do these programs log?

```js
const callback = x => console.log(x);

console.log('before');

new Promise(resolve => {
    console.log('hi');
    resolve(42);
}).then(callback);

console.log('after');
```

<div class="fragment fade-in">
```
before
hi
after
42
```
</div>

-------------------------------

Impossible to make 42 appear between *before* and *after*

You can convert sync -> Promise, but not Promise -> sync

-------------------------------

### Callbacks can do that!

Callback -> sync
```js
const callback = x => console.log(x);
console.log('before');
[42].forEach(callback);
console.log('after');
```
Just an artificial restriction!

-------------------------------

## Problem 3: Promises are eager, not lazy

-------------------------------

### What do you think will this program log?

```js

function fn(resolve, reject) {
    console.log('hello');
    //...
}

console.log('before');
new Promise(fn); //not using the promise at all
console.log('after');
```

-------------------------------

```
before
hello
after
```

-------------------------------

### There is no use for an eager Promise

Don't believe me?


-------------------------------

### You subconsciously make them lazy yourself

```js
const getMyData = () => {
    return new Promise(startXHR);
} //wrap Promise in function

const getUserFromData = () => {
    return getMyData() //unwrapping
        .then(data => data.user);
} //wrapping again
```

-------------------------------

### Resulting problem

Not composable any more (no `.then` on your wrapper), leads to constant wrapping/unwrapping in your code.

Why not like this:

```js
const getMyData = new Promise(startXHR);

const getUserFromData = getMyData.then(data => data.user);
```

-------------------------------

### Solution

Let promises have a `.run()` method, that uses the abstract description that is a promise and executes it.

```js
const getMyData = new Promise(startXHR);

const getUserFromData = getMyData.then(data => data.user);

//somewhere where needed
const user = getUserFromData.run();
```

-------------------------------

## Problem 2: Not cancleable

How long do you think this node script will run?

```js
Promise.race([
    new Promise(res => setTimeout(res, 1 * 1000)),
    new Promise(res => setTimeout(res, 10 * 1000)),
]);
```

-------------------------------

```
$ time -p node test.js
real 10,07
user 0,05
sys 0,01
```

-------------------------------

## The underlying problem

```js

const promiseA = getData(); //Promise is already running

const promiseB = getUserFromData(promiseA);
const promiseC = getOtherData(promiseA);

promiseB.cancel();
```

What should happen?

- cancel the whole chain?
- only promiseB gets canceled?
- Magic(TM)?

-------------------------------

## Possible solution: Refcounting

If you create a promise based on another promise, increase a counter. Cancel if counter drops to zero.

But: What if you dont want that?

-------------------------------

## Actual solution: Laziness (again)

```js
const getMyData = new Promise(startXHR);

const execution = getMyData.run()

execution.cancel();
```

-------------------------------

```js
function share(promise) {
    let executions = 0, ref = null;
    return {
        run() {
            if(executions === 0) { ref = promise.run(); }
            executions++;
        }
        cancel() {
            executions--;
            if(executions === 0){ ref.cancel(); }
        }
        /* ... stuff like then() */
    };
}

const promiseA = share(dataPromise);

const promiseB = promiseA.then(/* ... */);
const promiseC = promiseA.then(/* ... */);

promiseB.cancel(); //obvious that promiseA is still going

```

-------------------------------

# Why did no one thought about this?

Someone did point this out ... but was shut down because it was to much FP

-------------------------------

# Bonus round: Async/Await

A perfect example for an API built on top of promises

```js
async function foo() {
    let x = await Promise.resolve(42);
    console.log(x); //42
    return x;
} // async functions return a promise
```

-------------------------------

# Bonus round: Async/Await

Same problems as Promise (obviously)

But we can just *not* use promises inside an async function

-------------------------------

# Bonus round #2: Callback heaven

Andre Staltz created a spec for callbacks to create minimal observables/iteratables

-------------------------------

```js
const pausableInterval = period => (start, sink) => {
  if (start !== 0) return;
  let i = 0, id = null;
  const pause = () => {
    clearInterval(id);
    id = null;
  };
  const resume = () => {
    id = setInterval(() => { sink(1, i++); }, period);
  };
  sink(0, t => {
    if (t === 1) {
      if (id) pause();
      else resume();
    }
    if (t === 2) pause();
  });
  resume();
};
```

-------------------------------

# Better alternatives

>- Observables/Streams - Can deliver more than one value, lazy, not usable with async/await
>- Libraries like fluture - Implements a Future aka a proper Promise; lazy and async/await
>- Callbags - can model a stream or async iteratable
