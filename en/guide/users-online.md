<h3 id='users-online'>Users online count</h3>

This section details a full (small) application that tracks a users online
count using [Redis](http://redis.io). First up, you'll need to 
create a package.json file containing two dependencies: one for the redis
client, and another for Express itself. (Also, be sure you have redis installed
and running via `$ redis-server`.)

```js
{
  "name": "app",
  "version": "0.0.1",
  "dependencies": {
    "express": "3.x",
    "redis": "*"
  }
}
```

Next you'll need to create an app, and a connection to redis:

```js
var express = require('express');
var redis = require('redis');
var db = redis.createClient();
var app = express();
```

Next up is the middleware for tracking online users. Here we'll
use sorted sets, so that we can query redis for the users
online within the last N milliseconds. We do this by passing
a timestamp as the member's "score". Note that we're using the 
User-Agent string here, in place of what would normally be a user id.

```js
app.use(function(req, res, next){
  var ua = req.headers['user-agent'];
  db.zadd('online', Date.now(), ua, next);
});
```

This next middleware is for fetching the users online
in the last minute using **zrevrangebyscore**
to fetch with a positive infinite max value, so that we're
always getting the most recent users. It's capped with a minimum score
of the current timestamp minus 60,000 milliseconds.

```js
app.use(function(req, res, next){
  var min = 60 * 1000;
  var ago = Date.now() - min;
  db.zrevrangebyscore('online', '+inf', ago, function(err, users){
    if (err) return next(err);
    req.online = users;
    next();
  });
});
```

Finally we then use it, and bind to a port! That's
all there is to it. Visit the app in a few browsers
and you'll see the count increase.

```js
app.get('/', function(req, res){
  res.send(req.online.length + ' users online');
});

app.listen(3000);
```
