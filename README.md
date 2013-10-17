roz
===
Simple, expressive, whitelist authorization for express.js

![The Roz Headshot](roz-night-court.jpg) 

Why?
====
I wanted to glue authorization functions in front of my routes.  I didn't want RBAC, I just wanted to be
confident that rules were in front of my routes and that the default for a route was 403.

How?
====
Get a rozzed router by calling the wrap method on the express app:
```js
var roz = require("roz")();
var rozed = roz.wrap(app);
// Good idea to make sure you don't use the naked app by mistake
app = null
```

Now call any of the routing methods just like you would on app.  "namespace" The ```express-namespace``` module is
also supported.
```js
rozed.get("/posts", ...)
```

### The Roz Grammar

A rozed route supports a function-based grammar for building middleware rules. It's a good idea to just import all the grammar functions so your authorization rules read better:
```js
var roz = require("roz"),
    grant = roz.grant,
    where = roz.where,
    anyone = roz.anyone
```

Give anyone access to this route.  Authentication not even required.
```js
rozed.get( "/posts",
           roz( grant ( anyone )),
           ... )
```

Only let authenticated users create a post.
```js
var someone = function(req) { return req.isAuthenticated(); }

rozed.post( "/posts",
            roz( grant ( someone )),
            ... )
```

Use "where" to glue in a more specific rule.  For this example, only
an admin is allowed to edit posts.
```js
var actor = function(req) { return req.user; }
var isAdmin = function(user, cb) { cb(null, user.admin === true)};

rozed.patch( "/posts/:id",
             roz( grant( where ( isAdmin, actor, "id" ))),
             ... )
```

Only the an admin or the creator can delete a post.
```js

var isCreator = function(user, postId, cb) { ... };

rozed.delete( "/posts/:id",
              roz( grant( where ( isCreator, actor, "id" ))
                   grant( where ( isAdmin, actor ))),
              ... )
```


"revoke" can be used to deny access.
```js

var isCreator = function(user, postId, cb) { ... };

rozed.delete( "/posts/:id",
              roz( grant( where ( isCreator, actor, "id" ))
                   grant( where ( isAdmin, actor ))
                   revoke( where ( deletedToMuch, actor)) ),
              ... )
```

If you don't add a roz statement to a route, you'll get an exception
when you startup your express app.
```js
rozed.put( "/posts/:id",
           ... )
```
```bash
09:36:01 app  | Error: Roz: route does not include a roz statement: put /post/:id

```

### roz(grant|revoke [, grant|revoke*])
```roz``` expects one or more ```grant``` or ```revoke``` statements.  grant and revoke take a request
and callback with true (grant access) revoke (deny access) or null (unchanged).  Access is denied by
default.

### where(fn [, reqAccessor*])
```where``` is a helper that applies a function to variables extracted from the request.
The ```reqAccessors``` can either be a string or a function.  If it is a string, ```where``` will
call req.param(<string>).  If ```reqAccessor```, is a function it is called with the request and expected
to return a value.

Note: if you want to look in a location other than "req.param()", initialize roz like this:
```
var roz = require("roz")({lookin:"parsed"})
```
