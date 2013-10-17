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
var rozed = roz.wrap(app);
// Good idea to make sure you don't use the naked app by mistake
app = null
```

Now use it just like you would the router:
