roz
===
Simple, expressive and defensive authorization for express.js.

[<img src="https://raw.github.com/nomic/roz/master/roz-night-court.jpg"
     alt="The Roz Headshot"
     height="300px"/>](http://www.imdb.com/title/tt0086770/)

Why?
====
I wanted authorization functions at the front of my routes.  I didn't want RBAC
or anything opinionated about how my rules should be modeled, I just wanted:

* Clear rules without boiler plate
* A whitelist approach where the default is 403

Roz is about authorization, not authentication.  If you're not sure about the
difference between authentication and authorization, [wikipedia](http://en.wikipedia.org/wiki/Authentication#Authorization)
gives a good explanation.

How?
====
```js
var roz = require("roz")();  // The roz module is a callable -- call it

var rozed = roz.wrap(app);   // wrap your express app
app = null;  // Recommended to prevent accidental use

rozed.del( "/posts/:id",
           roz( grant( where ( isCreator, actor, "id" ))
                grant( where ( isAdmin, actor ))),
           ... );


function isAdmin(user, cb) { cb(null, user.admin === true) }
function isCreator(user, postId, cb) { ... }
```

*rozed* is a thin wrapper around *app*.  Call any of the express app routing
methods on it.  *namespace* from the *express-namespace* module is also supported.

If a grant does not fire, a 403 will be returned and the middleware after the
roz statement will not be called.  Notize that roz just provides the middleware
glue.  You implement custom rules in plain old javascript functions
like the *isAdmin* and *isCreator* examples.

Roz is defensive.  In addition to defaulting to 403, rozed routers demands that you
include at least one `roz` statement in any route you declare.  So, if you forget,
like this:

```js
rozed.get( "/posts/:id", doStuff )
```
You'll get an error when you try to start your express app:
```bash
09:36:01 app  | Error: Rozed route does not have a roz rule: get /posts/:id
```

### The Roz Grammar

Roz has a short, function-based grammar for inserting middleware rules. It's a good
idea to just import all the grammar functions so your authorization rules
read better:
```js
var roz = require("roz"),
    grant = roz.grant,
    revoke = roz.revoke, // Since "denied" is the default, you likely won't need this
    where = roz.where,
    anyone = roz.anyone
```

Here's how to give anyone, including unauthenticated users, access to a route.
```js
rozed.get( "/posts",
           roz( grant ( anyone )),
           ...);
```

If you only want to let authenticated users do something:
```js
rozed.post( "/posts",
            roz( grant ( someone )),
            ...);

function someone(req) { return req.isAuthenticated(); }
```

Note in the above you need to define *someone* yourself.  Roz is agnostic to
whatever your authentication scheme is so you'll need to provide that check.  As
above, that's generally pretty easy.

Also note that roz emits 403s.  If authentication is required and you want to
return a 401, you'll need to handle that before getting to the roz middleware.

Use `where` for more specific rules.  In the next example, only an admin
is allowed to edit posts.
```js
rozed.patch( "/posts/:id",
             roz( grant( where ( isAdmin, actor ))),
             ...);

function actor(req) { return req.user; }
function isAdmin(user, cb) { cb(null, user.admin === true) }
```

Here, only the admin or the creator can delete a post.
```js
rozed.del( "/posts/:id",
           roz( grant( where ( isCreator, actor, "id" ))
                grant( where ( isAdmin, actor ))),
           ...);

function isCreator(user, postId, cb) { ... }
```

If a `grant` fires, a subsequent `revoke` can flip authorization back
to denied.
```js
rozed.del( "/posts/:id",
           roz( grant( where ( isCreator, actor, "id" ))
                grant( where ( isAdmin, actor ))
                revoke( where ( deletedTooMuchAlready, actor ))),
           ...);
```

A Little More Detail
====================

### where(ruleFn [, reqAccessor*])
`where` applies a plain old javascript function, *ruleFn*, to data in the request.  You
specify the *ruleFn*, and then, for each argument,
a *reqAccessor* for getting data from the request.  If a *reqAccessor* is a string, `where`
will use it as an arg to *req.param()*.  If *reqAccessor* is a function, the function
is called with the request and expected to return a value.

Once extracted, the values are passed to the *ruleFn* along with a callback that
should be called with *true* or *false* (or an error).

### roz(grant(...)|revoke(...) [,grant(...)|revoke(...)*])
`roz` expects one or more `grant` or `revoke` statements.  Alternatively,
`grant` and `revoke` can be replaced by any function with this signature:
fn(req, callback), where the callback takes err then result.  The result
should be *true* (grant access), *false* (revoke access) or *null* (unchanged).

`grant` calls back with *true* or *null*, and `revoke` calls back with *false*
or *null*. Access is denied by default, so at least one *true* must be returned.

### require("roz")(options)
If you do not want to use *req.param()* to look up arguments for your where clauses,
you can specify an alternative location on *req*, e.g., *req.validated*, like this:
```
var roz = require("roz")({lookin:"validated"})
```

### More Questions?
Roz is about 100 lines of code.  Please give her a read.

