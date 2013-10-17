roz
===
Simple, expressive, white-list authorization for express.js

<em>Dang, you better roz that sh*t!</em>

![The Roz Headshot](roz-night-court.jpg) 

Why?
====
I wanted to glue authorization functions in front of my routes.  I didn't want RBAC, I just wanted to be
confident that rules were in front of my routes and that the default for a route was 403.

How?
====
Get a rozzed router by calling the wrap method on the express app:

```js
var roz = require("roz");
var rozed = roz.wrap(app);
// Good idea to make sure you don't use the naked app by mistake
app = null
```

Now call any of the routing methods just like you would on app.  The ```express-namespace``` module is
also supported.

```js
rozed.get("/posts", ...)
```

### The Roz Grammar

A rozed route supports a function-based grammar for building middleware rules.  It's a good idea to just
import all the grammar functions so your authorization rules read better:

```js
var roz = require("roz");
    grant = roz.grant, where = roz.where, anyone = roz.anyone, actor = roz.actor

...
rozed.get( "/posts",
           roz( where ( anyone )),
           fn )
...
var someone = function(req) { return req.isAuthenticated(); }

rozed.post( "/posts",
            roz( where ( someone )),
            fn )

...
var isAdmin = function(user) { return user.admin === true; }

rozed.post( "/posts/:id",
            roz( where ( isAdmin( actor ))),
            fn )

...
var isAdmin = function(user) { return user.admin === true; }

rozed.delete( "/posts/:id",
            roz( where ( isCreator( actor, "id" ))),
            fn )

```



roz( fn, [fn*] )
----------------
roz returns a middleware function that will return 403 or allow routing to proceed depending on whether or
not its argument functions grant's access.

grant( fn )

### roz
Let's grant anyone access to a route:

```js
rozed.get( "/user",
           roz( grant( anyone )),
           ... )
```

The ```roz``` function returns middleware.  It accepts a spl

