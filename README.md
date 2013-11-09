roz
===
Simple, expressive, whitelist authorization for express.js.

[<img src="https://raw.github.com/nomic/roz/master/roz-night-court.jpg"
     alt="The Roz Headshot"
     height="300px"/>](http://www.imdb.com/title/tt0086770/)

Why?
====
I wanted authorization functions at the front of my routes.  I didn't want RBAC
or anything opinionated about how my rules should be modeled, I just wanted:

* To make rules clear
* To be defensive: have a whitelist approach where default is 403
* To remove boilerplate (and silly mistakes)

Roz is about authorization, not authentication.  If you're not sure what the
difference between authentication and authorization is,
[read this](http://en.wikipedia.org/wiki/Authentication#Authorization).

How?
====
```js
var roz = require("roz")();  // The roz module is a callable -- call it
var rozed = roz.wrap(app);   // app is your express app

app = null  // Recommended to prevent accidental use

var isCreator = function(user, postId, cb) { ... };

rozed.del( "/posts/:id",
           roz( grant( where ( isCreator, actor, "id" ))
                grant( where ( isAdmin, actor ))),
           ... )

```

`rozed` is a thin wrapper around app.  Call any of the express app routing
methods on it.  `namespace` from the `express-namespace` module is also supported.

The rozed router demands that you include roz middleware in your routes.  So
if you forget:
```js
rozed.get( "/posts/:id", doStuff )
```
You'll get an error like this:
```bash
09:36:01 app  | Error: Rozed route does not have a roz expression: get /posts/:id
```

### The Roz Grammar

Roz has a function-based grammar for building middleware rules. It's a good
idea to just import all the grammar functions so your authorization rules
read better:
```js
var roz = require("roz"),
    grant = roz.grant,
    revoke = roz.revoke, // You likely won't need this one
    where = roz.where,
    anyone = roz.anyone
```

Here's how to give anyone, including unauthenticated users, access to a route.
```js
rozed.get( "/posts",
           roz( grant ( anyone )),
           ... )
```

If you only want to let authenticated users do something:
```js
var someone = function(req) { return req.isAuthenticated(); }

rozed.post( "/posts",
            roz( grant ( someone )),
            ... )
```

Note in the above you need to define `someone` yourself.  Roz is agnostic to
whatever your authentication scheme is so you'll need to provide that check.  As
above, that's generally pretty easy.

Also note that roz emits 403s.  If authentication is required and you want to
return a 401, you'll need to handle that before getting to the roz middleware.


Use `where` to glue in a more specific rule.  For this example, only
an admin is allowed to edit posts.
```js
var actor = function(req) { return req.user; }
var isAdmin = function(user, cb) { cb(null, user.admin === true)};

rozed.patch( "/posts/:id",
             roz( grant( where ( isAdmin, actor ))),
             ... )
```

Only the admin or the creator can delete a post.
```js

var isCreator = function(user, postId, cb) { ... };

rozed.del( "/posts/:id",
           roz( grant( where ( isCreator, actor, "id" ))
                grant( where ( isAdmin, actor ))),
           ... )
```

If a `grant` fires, a subsequent `revoke` can flip authorization back
to denied.
```js
rozed.del( "/posts/:id",
           roz( grant( where ( isCreator, actor, "id" ))
                grant( where ( isAdmin, actor ))
                revoke( where ( deletedTooMuchAlready, actor ))),
           ... )
```

A Little More Detail
====================

### roz(grant(...) | revoke(...) [grant(...) | revoke(...),* ] )
`roz` expects one or more `grant` or `revoke` statements.  `grant`
and `revoke` can be replaced with any funciton with this signature:
fn(req, callback), where the callback is your typical fn(err, result);

`grant` calls back with *true* (grant access) or *null* (unchanged),
and `revoke` calls back with *false* (revoke access) or *null* (unchanged).
Access will be denied by roz by default, so at least one `grant` must fire.

### where(ruleFn [, reqAccessor*])
`where` is a helper that applies a function (your authorization rule) to
variables extracted from the request.  Each *reqAccessor* can either be a
string or a function.  If it is a string, `where` will use it as an arg to
`req.param()`.  If *reqAccessor* is a function, the function is called with the
request and expected to return a value.

Once extracted, the values are passed to to the *ruleFn* with a callback which
should be called with true or false (or an error).

If you want to look up variables from a custom object on `req`, e.g.,
`req.validated`, and do not want to use `req.param()`, initialize
roz like this:
```
var roz = require("roz")({lookin:"validated"})
```

Roz is about 100 lines of code.  Give her a read if you've still got questions.

