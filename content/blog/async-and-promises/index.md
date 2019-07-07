---
title: Async and Promise.
date: "2019-07-07"
description: From Callback to Promise in JS 
---

_Content below comes from Kyle Simpson and his fantastic serie of books [you don't know JS](https://github.com/getify/You-Dont-Know-JS) and https://eloquentjavascript.net
If you're trying to strengthen your understanding of javascript as I do, I can't recommend enough to read those books._


In a ***synchronous*** programming model, things happen one at a time. When you call a function that performs a long-running action, it returns only when the action has finished and it can return the result. This stops your program for the time the action takes.

An ***asynchronous*** model allows multiple things to happen at the same time. When you start an action, your program continues to run. When the action finishes, the program is informed and gets access to the result (for example, the data read from disk).


Any time you wrap a portion of code into a function and specify that it should be executed in response to some event like _timer_,_mouse Click_, _Ajax response_ it creates a **later chunk**.

Generally speaking, anything computation-related is sync and anything input/output/timing-related is async

But how JS engine handles both async and synchronous code ? 

#### Event Loop

JS engine runs inside a _hosting environment_, being the server for Node, or the Web Browser. All of those environment have a mechanism to handle executing multiple chunks of your program over time called the **event loop**

The JS Engine runs the synchronous code first. Any asynchronous code is sent to the **task** or **callback** queue. When the callStack from the JSEngine is empty - the eventLoop gets the next item on the task/callback queue and execute it. 

Picture below comes from this excellent talk : https://www.youtube.com/watch?v=8aGhZQkoFbQ from Philip Roberts

![img](https://cdn-images-1.medium.com/max/1600/1*pIEFRBvMqxpDipMnqkVprA.jpeg)



What happens if there are already 20 items in the event loop ? Your callback waits, it gets behind the others. This explains why _setTimeout()_ timers are a minimal timer not a guaranted one.

#### Jobs

As of ES6, there is a new concept layered on top of the event loop : **job queue**. It's a mechanism used by Promises that allow them to stack some async-implied action at the end of this tick in the eventLoop. Meaning those async action won't be stacked at the end of the task queue but rather executed just after the current tick of the event loop. 

#### Callbacks

From a strictly asynchronous point of view, a **callback** is a function that is to be executed after another function has finished executing, because it serves as the target for the event loop to **call back into** the program whenever that item in the queue is processed

According to Kyle - there are 2 major drawbacks to callback : 

- difficult to reason about because of their nonlinear/nonsequential ways vs our sequentional brain. Hard to reason code is bad code that leads to bug
- Callbacks suffer from inversion of control in that they implicitly give control over to another party to invoke the continuation of your program.

Those 2 main problem have led JS to deliver **promise** in ES6



#### Promises

The ECMA Committee defines a promise as

> A Promise is an object that is used as a placeholder for the eventual results of a deferred (and possibly asynchronous) computation.



Simply, **a promise is a container for a future value.** 



```javascript
//promise creation 
const myFirstPromise = new Promise((resolve,reject) =>  resolve( 42))

//Promise object accept one function as argument which itself takes 2 callbacks : resolve and reject

//you can also directly call resolve or reject on the Promise object
const myFirstRejectedPromise = Promise.reject(new Error('fail'))
const myFirstResolvedPromise = Promise.resolve('sucess')
```



Promise caracteristics :

- **Resolved once** and only once. Once it has been resolved (either fullfilled or rejected) it becomes an immutable value 
- Promise by definition **are always asynchronous**. Even an immediately fulfilled Promise like `new Promise(function(resolve){resolve(42)})` cannot be observed synchronously. When you call then on a promise the callback you provide will always be called asynchronously
- Nothing can prevent a promise from notifying you of its resolution.  If you register both fullfilment and rejection callbacks for Promise, and the promise gets resolved, **one of the two callbacks will always be called.**



```javascript
// 1 - Resolved Once from https://codeburst.io/a-simple-guide-to-es6-promises-d71bacd2e13a
const myPromise = new Promise((resolve, reject) => {
    if (Math.random() * 100 < 90) {
        console.log('resolving the promise ...'); // res
        resolve('Hello, Promises!');
    }
    reject(new Error('In 10% of the cases, I fail. Miserably.'));
});

// Two functions 
const onResolved = (resolvedValue) => console.log(resolvedValue);
const onRejected = (error) => console.log(error);

myPromise.then(onResolved, onRejected);
myPromise.then(onResolved, onRejected);

// Output (in 90% of the cases)

// resolving the promise ... - only once here as its only resolved once
// Hello, Promises!
// Hello, Promises! - resolved value is cached

```



####Chaining Promise

```javascript
var resolved = Promise.resolve( 'success' );
resolved.then(fulfilledValue => console.log('fullfill : ', successValue), errorValue => console.log('rejected:', errorValue)) // resolved : success

var rejected = Promise.reject( 'error' );
rejected.then(fulfilledValue => console.log('fullfill : ', successValue), errorValue => console.log('rejected:', errorValue)) // rejected : error

//then accepts 2 callback, first is for success scenario, the second is for error. Both takes the fulfilled value as argument, either success or error

```

- Every time you call `then()`on a Promise, it creates and returns a new Promise which we can chain with. 

- Whatever value you return from the `then()`call fullfillment callback `the first parameter` is automatically set as the fullfilment of the chained Promise
- if you don't return an explicit value , an implicit `undefined` is assumed and the promise still chain together the same way. Each promise resolution is thus just a signal to proceed to the next step

Below we wrapped `42` up in a promise that we returned, it still got unwrapped and ended up as the resolution of the chained promise, such that the second `then(..)` still received `42`. If we introduce asynchrony to that wrapping promise, everything still nicely works the same:

```javascript
var p = Promise.resolve( 21 );

p.then( function(v){
	console.log( v );	// 21

	// create a promise to return
	return new Promise( function(resolve,reject){
		// introduce asynchrony!
		setTimeout( function(){
			// fulfill with value `42`
			resolve( v * 2 );
		}, 100 );
	} );
} )
.then( function(v){
	// runs after the 100ms delay in the previous step
	console.log( v );	// 42
} );
```

Now we can construct a sequence of however many async steps we want, and each step can delay the next step (or not!), as necessary.

If you call `then` on a promise and you only pass a fulfillment or a rejection handler, the counterpart is assumed.

```javascript
var p = new Promise( function(resolve,reject){
	reject( "Oops" );
} );

var p2 = p.then(
	function fulfilled(){
		// never gets here
	}
	// assumed rejection handler, if omitted or
	// any other non-function value passed
	// function(err) {
	//     throw err;
	// }
);

var p = Promise.resolve( 42 );

p.then(
	// assumed fulfillment handler, if omitted or
	// any other non-function value passed
	// function(v) {
	//     return v;
	// }
	null,
	function rejected(err){
		// never gets here
	}
);
```



#### Terminology

Some distinctions to do between **resolve**, **fulfill** and **reject** which change between constructor call and `.then`

The first callback parameter of the `Promise(..)` constructor will unwrap either a thenable (identically to `Promise.resolve(..)`) or a genuine Promise:

```javascript
var rejectedPr = new Promise( function(resolve,reject){
	// resolve this promise with a rejected promise
	resolve( Promise.reject( "Oops" ) );
} );

rejectedPr.then(
	function fulfilled(){
		// never gets here
	},
	function rejected(err){
		console.log( err );	// "Oops"
	}
);
```

resolve can be called to resolve a  `rejected` promise. Therefore `resolve(..)` is the appropriate name for the first callback parameter of the `Promise(..)`constructor

Whereas in the case of the first parameter to `then(..)`, it's unambiguously always the fulfillment case, so there's no need for the duality of "resolve" terminology as the .then chains on a resolved value/

**Warning:** The previously mentioned `reject(..)` does **not** do the unwrapping that `resolve(..)` does. If you pass a Promise/thenable value to `reject(..)`, that untouched value will be set as the rejection reason. A subsequent rejection handler would receive the actual Promise/thenable you passed to `reject(..)`, not its underlying immediate value.



####Error Handling

If at any point in the creation of a Promise, or in the observation of its resolution, a JS exception error occurs, such as a `TypeError` or `ReferenceError`, that exception will be caught, and it will force the Promise in question to become rejected.

For example:

```javascript
var p = new Promise( function(resolve,reject){
	foo.bar();	// `foo` is not defined, so error!
	resolve( 42 );	// never gets here :(
} );

p.then(
	function fulfilled(){
		// never gets here :(
	},
	function rejected(err){
		// `err` will be a `TypeError` exception object
		// from the `foo.bar()` line.
	}
);
```

The JS exception that occurs from `foo.bar()` becomes a Promise rejection that you can catch and respond to.

But what happens if a Promise is fulfilled, but there's a JS exception error during the observation (in a `then(..)`registered callback)? Even those aren't lost, but you may find how they're handled a bit surprising, until you dig in a little deeper:

```javascript
var p = new Promise( function(resolve,reject){
	resolve( 42 );
} );

p.then(
	function fulfilled(msg){
		foo.bar();
		console.log( msg );	// never gets here :(
	},
	function rejected(err){
		// never gets here either :(
	}
);
```

Wait, that makes it seem like the exception from `foo.bar()` really did get swallowed. Never fear, it didn't. But something deeper is wrong, which is that we've failed to listen for it. The `p.then(..)` call itself returns another promise, and it's *that* promise that will be rejected with the `TypeError` exception.



From https://javascript.info/promise-error-handling :

The code of a promise executor and promise handlers has an "invisible `try..catch`" around it. **If an error happens, it gets caught and treated as a rejection.**

For instance, this code:

```javascript
new Promise((resolve, reject) => {
  throw new Error("Whoops!");
}).catch(alert); // Error: Whoops!
```

…Works exactly the same as this:

```javascript
new Promise((resolve, reject) => {
  reject(new Error("Whoops!"));
}).catch(alert); // Error: Whoops!
```

The "invisible `try..catch`" around the executor automatically catches the error and treats it as a rejection.

This happens not only in the executor, but in its handlers as well. If we `throw` inside a `.then` handler, that means a rejected promise, so the control jumps to the nearest error handler.

Here’s an example:

```javascript
new Promise((resolve, reject) => {
  resolve("ok");
}).then((result) => {
  throw new Error("Whoops!"); // rejects the promise
}).catch(alert); // Error: Whoops!
```

This happens for all errors, not just those caused by the `throw` statement. For example, a programming error:

```javascript
new Promise((resolve, reject) => {
  resolve("ok");
}).then((result) => {
  blabla(); // no such function
}).catch(alert); // ReferenceError: blabla is not defined
```

The final `.catch` not only catches explicit rejections, but also occasional errors in the handlers above.



#### **Active Retrieval**

To end this article, here are a few questions you can ask/challenge yourself to make sure you have understood/ the core concept. I would suggest to even come back in a week to those questions and ask them again.  

- What are the characteristics of promise ? 
- What are the different ways to create promise in JS ? 
- What happens if a .then miss one of its callback ?
- What happens if an error happens in the callback from the `new Promise` syntax ? 
- What's the .catch equivalent to ? 