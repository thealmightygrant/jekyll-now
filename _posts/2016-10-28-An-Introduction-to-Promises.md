---
layout: post
title: An Introduction to Promises
permalink: An-Introduction-to-Promises
tags: javascript, promises, callbacks
---

Promises, similar to [callbacks](http://callbackhell.com) provide a method for asynchronous programming in javascript. Node.js is a single threaded environment. So, if you have a long running instruction, for instance, requesting data from somewhere outside of your application (like an API or a database), then you should be using some method to ensure that while you are waiting on this data, your application can continue processing other instructions. Promises are one such possiblity.

# Promises, who needs them?

While working in javascript these past two years, I have been blissfully unaware of promises. Apparently, this is one of the recent great debates surrounding the language. Don't fall into the rabbit hole of looking into this debate too deeply. Really, the idea behind promises is to avoid the cascade of functions containing other functions containing other functions ad infinitum that can result from any decently complex javascript project.

Let's say that you want to get some data about a user from a few services like Github, Twitter, and Instagram and combine that data to create a profile about a person. Then in a naive implementation, you would need to hit up each service and wait for them to return before creating that profile.

Here's some callbacks that will do just that:

```js
getData("api.github.com/users/" + gh_username, function(error, gh_data){
    if(error)
        err_callback(error);
    getData("api.twitter.com/users/" + tw_username, function(error, tw_data){
        if(error)
            err_callback(error);
        getData("api.instagram.com/users/" + inst_username, function(error, inst_data){
            if(error)
                err_callback(error);
            createProfile(username, gh_data, tw_data, inst_data);
        })
    })
})
```

Even with just three callbacks, this code is starting to get a little hairy. Some libraries like [async.waterfall](http://caolan.github.io/async/) would clear this up nicely so that it's not as ugly.

However, the real issue here is not what it's doing, but how it's doing it. What if we added a call to the end of the above code to display the created profile?

```js
getData(..., function(error, gh_data){
    ...
    createProfile(username, gh_data, tw_data, inst_data)
})
displayProfile(username)
```

We could intelligently design the `displayProfile` function to not crap out if/when it gets called before there's any data in the profile, but do we really want to expect the user to refresh their browser in a couple of seconds? Or should we put a timeout or a flag on this function to ensure that when data is available, it can be called again?

Dominic Denicola calls this ["continuation passing style"](http://www.slideshare.net/domenicdenicola/callbacks-promises-and-coroutines-oh-my-the-evolution-of-asynchronicity-in-javascript). In a sense, this means that anything done with data acquired by a callback needs to be done inside of another callback.

```js
getData(..., function(error, gh_data){
    ...
    createProfile(username, gh_data, tw_data, inst_data, function(){
        //we know the profile is there now
        displayProfile(username)
    })
})

//profile exists here? probably not
```

What the program really needs to know from an asynchronous function is whether that function has succeeded, failed, or is still in progress and what data or errors resulted from it. This leads to the concept of promises. An encapsulation of an asynchronous function in an object that can give information about both the function's status (running, finished, errored) and the data or error that resulted from it. Upon completion, instead of a callback function being *required* to be passed in as an argument, the function instead returns a promise. **There is no requirement to know what it will be doing next with the data.** *Function definitions become much simpler and more flat by requiring fewer functions as arguments.* If something needs data from an asynchronous function, then it simply uses/reuses the promise.

So, back to our example, with Promises this same set of calls could be written like this:

```js
//convert from callbacks to promises
Promise.promisify(getData);
Promise.promisify(createProfile);

//NOTE: getData is now a promise returning function
//      not a promise, a function that returns a promise

var gh_promise = getData("api.github.com/users/" + gh_username)
var tw_promise = getData("api.twitter.com/users/" + tw_username)
var inst_promise =getData("api.instagram.com/users/" + inst_username)

return Promise.join(
    gh_promise,
    tw_promise,
    inst_promise
).then(function(gh_data, tw_data, inst_data){
    return createProfile(username, gh_data, tw_data, inst_data)
}).then(function(){
    displayProfile(username)
}).catch(function(err){
    console.error(err)
    throw err;
})
```

Promises are awesome. I'm a huge fan. It can be tough to see why you need them at first, but once you are in tune with using them, they will be your best friend. Check out these links below and in the article. Honestly, it probably took me two days to use them, and a month to use them correctly. Don't let my experience dissuade you too much, they are a breath of fresh air. Don't dive too deeply too quickly (like i did), and you will be up and using them in no time.

[an awesome tutorial on promises](https://promise-nuggets.github.io/),
[promises are fast and memory efficient](https://spion.github.io/posts/why-i-am-switching-to-promises.html), and
[bluebird.js, the fastest, most memory efficient, and somewhat well documented promise library.](http://bluebirdjs.com/)

Take it easy,<br><br>
Grant
