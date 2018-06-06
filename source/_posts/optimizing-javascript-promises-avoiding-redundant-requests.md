---
title: Optimizing javascript promises by avoiding redundant requests
permalink: optimizing-javascript-promises-avoiding-redundant-requests
date: 2015-12-10
tags:
excerpt: Sometimes when working with asynchronous JavaScript we unnecessary send the same HTTP request multiple times from different places. In this article we'll discuss a technique to avoid these redundant calls and make our promises more efficient, and will present a tool to apply this to our existing code without effort. 
---
Sometimes when working with asynchronous JavaScript we unnecessary send the same HTTP request multiple times from different places. In this article we'll discuss a technique to avoid these redundant calls and make our promises more efficient, and will present a tool to apply this to our existing code without effort. 
<!-- less --> 
**[Update 11 May 2016](#upd110516)**: The ES2015-Promises version is available as an [npm package][9].

When working with asynchronous requests in JavaScript we often have the case where a function that returns a promise may be called from a number of different places, from 0 to many times.

This is especially the case in SharePoint, where a typical scenario is having a page with multiple web parts -or multiple instances of the same one- that need to fetch some data asynchronously via JSOM or REST API and do something with it.

Since the AJAX call is costly, why not make the trip to the server for the data just once and then reuse it trough the lifetime of the page?

 *Note: This article assumes that the data we are retrieving is rather static -for example, information on the current user's User Profile or a term set- and it is reasonable to keep and reuse a copy through the lifetime of the page.*

## Caching the data

An initial implementation of this using [jQuery Deferred objects][1] could be something like the following. In this example the AJAX operation will be requesting the current page, passing in a `tag` parameter so we can identify each call in the timeline:

```javascript
var fetchData = (function() {
  var cachedData = undefined;

  return function(tag) {
    var dfd = $.Deferred();

    if (cachedData !== undefined) { 
      dfd.resolve(cachedData);
    } 
    else {
      //This could also be multiple nested AJAX calls
      $.get("?" + tag).then(
        function(result) {
          cachedData = result;
          dfd.resolve(cachedData);
        },
        function() {
          dfd.reject(arguments);
        }
      );
    }
    return dfd.promise();
  };
})();
```
 In this example, after the data is retrieved for the first time, it is saved into a `cachedData` variable, and subsequent calls will make use of it. Note that the `cachedData` variable is left outside of the inner function (i.e. in its [closure][2]), so it is shared across multiple calls.

The problem with this solution arises when calls are made to the function before the first one has returned (and this will often be the case with web parts calling some initialization function virtually at the same time, when the page is being loaded). In this case every additional call will make an unnecessary extra trip to the server, rendering the solution pretty much useless. 

![fetchData](./fetchData.png)

Here we can see how calling the function 3 times in a row results in 3 separate AJAX requests being fired. Once any of them is completed and `cachedData` populated, the fourth call uses it as expected.

## Solving the concurrency problem

To overcome this, a single deferred object can be used for all calls, which we will store in the closure in the same way we did earlier with `cachedData`.

```javascript
var fetchDataBetter = (function() {
  var dfd = null;

  return function(tag) {
    if (!dfd) {
      dfd = $.Deferred();

      //This could also be multiple nested AJAX calls
      $.get("?" + tag).then(
        function(result) {
          //Potentially do some data transformation on result
          dfd.resolve(result);
        },
        function() {
          dfd.reject(arguments);
        }
      );
    }
    return dfd.promise();
  };
})();
```
The first call to this function will find that `dfd` is null so it will be initialised as a `$.Deferred` object, the AJAX request sent, and the promise returned. Subsequent calls will find `dfd` already defined, and will just return the promise.

Now the good thing with promises is that they handle automatically the "queuing" process:

- If, at the time of calling `fetchDataBetter("x").then(/*...*/)` the promise is still pending (i.e. the AJAX request hasn't returned yet), it will automatically queue until it is fulfilled (succeeded) or rejected, and then the promises mechanism will automatically take care of calling the callbacks specified in `then()`. 
- On the other hand if the promise has already been fulfilled/rejected, the `then()` section will run immediately, preserving the value with which it was resolved/rejected. This explains why the `cachedData` variable is no longer necessary when using this approach.

Here's an example of this in action:

![fetchDataBetter](./fetchDataBetter.png)

In this case, even though the function was called 3 times in a row, only the first one resulted in a request, the other 2 being queued until it finished. The fourth call just made use of the already resolved value.

So far so good. Now we have a technique we can use in our data-retrieving functions to do the hard work only once, but every time we use it we have to write all this extra code which really doesn't add any functional value. If only we could automate this.

## The `_.once` function

JavaScript libraries like [underscore][3] and [lodash][4] incorporate the `_.once` utility function:

> `_.once(func)` - Creates a function that is restricted to invoking `func` once. Repeat calls to the function return the value of the first call.

Let's see it in action with an example:

![_.once](./once.png)

As with `fetchDataBetter`, this makes `func()` only run the first time and reuse the result on further calls, but in this case our `func()` doesn't need to be aware of it - The "run once then reuse" magic is automatically handled by wrapping the function with `_.once` and using the *once'd* version instead.

Applying the same idea to our method, we can implement a similar utility that works with promises and add it to our utility library to use whenever we want.

## Introducing the *ensurizer*

This is what our wrapper function looks like:

```javascript
// Wraps a promise callback in another promise function so that 
// multiple calls to the wrapper will result in a single call 
// of the original one, the rest being enqueued and called when it is resolved
var ensurize = function(promiseCallback) {
  var dfd = null;
  return function() {
    if (!dfd) {
      dfd = $.Deferred();
      promiseCallback.apply(this, arguments).then(dfd.resolve, dfd.reject);
    }
    return dfd.promise();
  };
};
```

For lack of a better idea, I named the function `ensurize` because it converts a function like `loadData` function into an `ensureDataLoaded`. This *ensure* concept meaning "make sure something is done, otherwise do it" is present in internal APIs from Microsoft themselves, like [`Controls.EnsureChildControls()`][5], [`SPWeb.EnsureUser`][6], [`EnsureScriptFunc()`][7] and so on. It could have also be named `promiseOnce()` for consistency with its cousin `_.once()`. Opinions as well as other suggestions are welcome :)

It can be used like this:

```javascript
var loadSomeData = function(tag) {
  return $.get("?" + tag);
};
var ensureSomeDataLoaded = ensurize(loadSomeData);

//Now call ensureSomeDataLoaded(x) multiple times
```

Note how the `loadSomeData` function doesn't know anything about the optimisation - In fact it can still be called on its own. This also means we can easily *ensurize* any function as long as it returns a promise. And which asynchronous function doesn't return one these days?

The *ensurized* function accepts any number of parameters, which will be passed into the inner function in the first (and only) time it will be called.

Note also that each call to `ensurize` will generate a new wrapper even for the same inner function. This means if we need to call it with different parameters, or force to refresh a copy of the data, we can just create a new copy of it.

So now it's time for you to include it in your utility toolbelt. Please don't contaminate the global scope, include it under your own *namespace*. You can also add it to `Function.prototype` for syntactic sugar purposes (although I'm not a fan of messing around with the prototype of JavaScript's native types):

```javascript
Function.prototype.ensurize = function () { return ensurize(this) };
var ensureSomeDataLoaded = loadSomeData.ensurize();
```

Please leave a comment if you found the post interesting (or not). In future posts I will provide some real-life examples of this technique in use, so stay tuned!

<a name="upd110516" id="upd110516"></a>

## Update 11 May 2016: NPM package

I went on and implemented the *ensurize* idea using [ES2015 Promises][8], which is going to be the standard that replaces jQuery Deferred objects. The source code is [available on GitHub][10] and is pretty much this function:

```javascript
function ensurize(promiseCallback) {
    var promise = null;
    return function () {
        var that = this,
            args = arguments;
        if (!promise) {
            promise = new Promise(function (resolve, reject) {
                promiseCallback.apply(that, args).then(resolve, reject);
            });
        }
        return promise;
    };
}
```

You can get the [ensurize package from npm][9].

[1]: https://api.jquery.com/jQuery.Deferred/
[2]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures
[3]: http://underscorejs.org/#once
[4]: https://lodash.com/docs#once
[5]: https://msdn.microsoft.com/en-us/library/system.web.ui.control.ensurechildcontrols
[6]: https://msdn.microsoft.com/en-us/library/microsoft.sharepoint.spweb.ensureuser.aspx
[7]: http://www.eliostruyf.com/correctly-including-scripts-display-templates/
[8]: https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Promise
[9]: https://www.npmjs.com/package/ensurize
[10]: https://github.com/spcfran/ensurize