Action.js, a sane way to write async code
=========================================

Action.js offer a [faster](https://github.com/winterland1989/Action.js/wiki/Benchmark) and [simpler](~200LOC) alternative to [Promise](http://promisesaplus.com), got 5 minutes?

Understand Action.js in 5 minutes
---------------------------------

Suppose we want to solve the nest callback problem form scratch, there's an async function called `readFile`, and we want to use it to read `data.txt`, we have to supply a `callback` to it:

    readFile("data.txt", callback)

Instead we don't give a callback to it right now, we save this read action in a new `Action`:

```js
var Action = function Action(action1) {
    this.action = action1;
}

var readFileAction = new Action(
    function(cb){
        readFile("data.txt", cb);
    }
);
```
We have following objects on our heap:

    +----------------+----------+
    | readFileAction | .action  | 
    +----------------+----+-----+
                          |
                          v
                    +----------+---------------+
                    | function | cb            |
                    +----------+---------------+   
                    | readFile("data.txt", cb) |
                    +--------------------------+

Ok, now we must have a way to extract the action from our `readFileAction`, instead of using `readFileAction.action` directly, we write a function to accpet a callback, and pass this callback to the action inside our `readFileAction`:

```js
Action.prototype._go = function(cb) {
    return this.action(cb);
};
readFileAction._go(function(data){
    console.log(data);
})
```

You should understand what above `_go` does is equivalent to following:

```js
readFile("data.txt", function(data){
    console.log(data);
});
```

Just with one difference, we seperate action creation(wrap `readFile` in `new Action`) and application(use `_go` to supply a callback), in fact we have successfully did a [CPS transformation](https://en.wikipedia.org/wiki/Continuation-passing_style), we will talk about that later.

Now we want to chain more callbacks in Promise `then` style:

```js
Action.prototype._next = function(cb) {
    var self = this;
    return new Action(function(_cb) {
        return self.action(function(data) {
            var _data = cb(data);
            return _cb(_data);
        });
    });
};
```

Let's break down `_next` a little here:

+ `_next` accept a callback `cb`, and return a new `Action`.

+ When the new `Action` fired, the original `Action`'s action will be fired first, and send the value to `cb`.

+ we save the `_data` produced by `cb`, and wait for a future `_cb`.

With our `_next`, we can chain multiply callbacks and pass data between them:

```js
readFileAction
._next(function(data){
    return data.length;
})
._next(function(data){
    // data here is the length we obtain last step
    console.log(data);
    return length > 0
})
._go(function(data){
    // data here is a Boolean
    if(data){
        ...
    }
})
```

If we want to present it with diagram, it should look like this:

    +----------------+----------+
    | ActionTwo      | .action  | 
    +----------------+----+-----+
                          |
                          v
                    +----------+----------------+
                    | function | cb_            |
                    +----------+----------------+  
                    | cb = function(data){      |
                    |   console.log(data);      |
                    |   return length > 0       |
                    | }                         |
               +--- + ActionOne.action(         |
               |    |   function(data){         |
               |    |     cb_(cb(data))         |
               |    |   });                     | 
               |    +---------------------------+
               |                            
               v       
    +----------------+----------+            
    | ActionOne      | .action  | 
    +----------------+----+-----+
                          |
                          v
                    +----------+----------------+
                    | function | cb_            |
                    +----------+----------------+  
                    | cb = function(data){      |
                    |   return data.length      |
                    | }                         |
               +--- + readFileAction.action(    |
               |    |   function(data){         |
               |    |     cb_(cb(data))         |
               |    |   });                     | 
               |    +---------------------------+
               |           
               v          
    +----------------+----------+
    | readFileAction | .action  | 
    +----------------+----+-----+
                          |
                          v
                    +----------+---------------+
                    | function | cb            |
                    +----------+---------------+   
                    | readFile("data.txt", cb) |
                    +--------------------------+

`ActionOne` and `ActionTwo` are `Action`s first and second `_next` returned respectively, Now if we give `ActionTwo` a `callback` with `_go`, the whole callback chain will be fired sequential.

Nice, we just use a simple class with only one field, two very simple functions, the callbacks are now written in a much more readable way, but we have a key problem to be solved yet: what if we want to nest async `Action`s inside an `Action`, it turn out with an adjusted `_next` function, we can handle that:

```js
Action.prototype._next = function(cb) {
    var self = this;
    return new Action(function(_cb) {
        return self.action(function(data) {
            var _data = cb(data);
            if (_data instanceof Action) {
                return _data._go(_cb);
            } else {
                return _cb(_data);
            }
        });
    });
};
```

We use `instanceof Action` to check if a callback returns a `Action` or not, if an `Action` is returned, we fire it with callbacks in future:

```js
readFileAction
._next(function(data){
    var newFile = parse(data);
    return new Action(function(cb){
        readFile(newFile, cb);
    });
})
._go(function(data){
    // data here is the newFile's content
    console.log(data)
})
```

Now we can say we have solved the callback hell problem! Well, actually just 50% of it.
Before we proceed another 50%, one important thing to keep in mind: **an `Action` is not a `Promise`, it will not happen if you don't fire it with `_go`, and it can be fired multiple times, it's just a reference to a wrapped function**:

```js
readFileAction
._next(processOne)
._go(console.log)

// after we do other things, or inside another request handler
...

// processTwo may receive different data since the file may change!
readFileAction
._next(processTwo)
._go(console.log)
```

I'll present `Action.freeze` in [Difference from Promise](https://github.com/winterland1989/Action.js/wiki/Difference-from-Promise) to give you Promise behavior when you need it, now let's attack another 50% of the callback hell issue.

Error handling
--------------

One biggest issue with `Promise` is that error handleing is somewhat magic and complex:

+ It will eat your error sliently if you don't supply a `catch` at the end of the chain.

+ You have to use two different functions, `resolve` to pass value to the callbacks and `reject` to skip them, what will happen if you `throw` an `Error`, well, just the same as `reject`

What we can do to make it simpler? It's a complex problem, so we start solving it by simplify it: **We use `Error` type as a special type to pass error information to the downstream**, what does this mean?

```js
Action.prototype.next = function(cb) {
    var self = this;
    return new Action(function(_cb) {
        return self.action(function(data) {
            if (data instanceof Error) {
                return _cb(data);
            } else {
                var _data = cb(data);
                if (_data instanceof Action) {
                    return _data._go(_cb);
                } else {
                    return _cb(_data);
                }
            }
        });
    });
};
```

Here, let me present the final version of our `next` function, comparing to `_next` we write before, can you see what's the different? It still reture a new `Action`, when it fired, the original action are called, and we checked if the data are `instanceof Error`, if it's not, everything as usual, we feed it to `cb` that `next` received, but if it's an `Error`, we pass it to a future `_cb`, which we don't have now.

Symmetrically, we have to define a function that special deal with `Errors`, and let normal values pass:

```js
Action.prototype.guard = function(cb) {
    var self = this;
    return new Action(function(_cb) {
        return self.action(function(data) {
           if (data instanceof Error) {
            var _data = cb(data);
                if (_data instanceof Action) {
                    return _data._go(_cb);
                } else {
                    return _cb(_data);
                }
            } else {
                return _cb(data);
            }
        });
    });
};
```

This time, we know the `cb` that `guard` received are prepared for `Error` values, so we flip the logic.

Following code demonstrate how to use our `next` and `guard`:

```js
new Action(function(cb){
    readFile('fileA', function(err, data){
        if (err){
            // see how to pass an Error to downstream, not reject, not throw, just return
            cb(err);
        }else{

            cb(data);
        }
    });
})
.next(function(data){
    return processData(data);
})
.next(function(data){
    return new Action(function(cb){
        processDataAsync(data, cb);
    })
})
.next(
    try{
        return someProcessMayWentWrong(data);
    }catch(e){
        // same as above, we return the error
        return e;
    }
}))
.next(function(data){
    // This process will be skip if previous step pass an Error
    return anotherProcess(data);
})
.guard(function(e){
    // This process will be skip if there's no Errors
    return processError(e);
});
._go(console.log);

```

The final result will be produced by `anotherProcess` if `someProcessMayWentWrong` didn't go wrong, or produced by `processError` otherwise.

You can place `guard` in the middle of the chain, all `Errors` before if will be handled by it, and the value it produced, will be passed to the rest of the chain.

So, what if the use didn't supply a `guard`? Well, since use have to supply a callback to the `_go`, they can check if the callback they supplied received an `Error` or not like this:

```js
apiReturnAction('...')._go(function(data){
    if (data instanceof Error){
        //handle error here
        ...
    } else {
        // process data here
        ...
    }
});

```
Yeah, it does work, but we don't want force our user to write like above, and we should throw `Error` in case user didn't `guard` them:

```js
Action.prototype.go = function(cb) {
    return this.action(function(data) {
        if (data instanceof Error) {
            throw data;
        } else if (cb != null) {
            return cb(data);
        }
    });
};

```

Now if user don't guard `Error`s, we will yell at them when `Error` occurs!

```js
new Action(function(cb){
    readFile('fileA', function(err, data){
        if (err){
            // suppose we got an Error here
            cb(err);
        }else{

            cb(data);
        }
    });
})
.go() // The Error will be throw!

```

Finally, to ease error management, and to attack the [v8 optimization problems](https://github.com/petkaantonov/bluebird/wiki/Optimization-killers#2-unsupported-syntax). We recommand use `Action.safe`:

```js
// this small function minimize v8 try-catch overhead
// and make attaching custom Error easy
Action.safe = function(err, fn) {
    return function(data) {
        try {
            return fn(data);
        } catch (_error) {
            return _error;
        }
    };
};
```

And use `safe` wrap your `someProcessMayWentWrong` like this:

```js
var safe = Action.safe;
new Action(function(cb){
    readFile('fileA', function(err, data){
        if (err){
            // see how to pass an Error to downstream, not reject, not throw, just return
            cb(err);
        }else{
            cb(data);
        }
    });
})
.next(
    safe(new Error("PROCESS_ERROR_XXX: process xxx failed when xxx"), someProcessMayWentWrong)
)
.next(...)
.next(...)
.guard(function(e){
    if (e.message.indexOf('ENOENT') === 0){
        ...
    }
    if (e.message.indexOf('PROCESS_ERROR_XXX') === 0 ){
        ...
    }
})
.go()

```

That's all core functions of `Action` is going to give you, hope you enjoy my solution :), Check [API doc](https://github.com/winterland1989/Action.js/wiki/API-document) for more interesting things like `Action.parallel`, `Action.race`, `Action.sequence` and `Action.retry`, It's also highly recommend to read [Difference from Promise](https://github.com/winterland1989/Action.js/wiki/Difference-from-Promise) to get a deeper understanding.
