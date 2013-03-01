data.io
===

Bidirectional data syncing via Socket.IO

Example
---

On the server:

```javascript
var io = require('socket.io').listen(3000);
var data = require('data.io')(io);

var messages = data.bucket('messages');

var store = {};
var id = 1;

messages.use('create', 'update', function(req, res) {
    var message = req.data;
    if (!message.id) message.id = id++;
    store[message.id] = message;
    res.send(message);
});

messages.use('delete', function(req, res) {
    var message = store[req.data.id];
    delete store[message.id];
    res.send(message);
});

messages.use('read', function(req, res) {
    var message = store[req.data.id];
    res.send(message);
});
```

On the client:

```html
<script src="/socket.io/socket.io.js"></script>
<script src="/data.io.js"></script>

<script>
    var socket = data(io.connect());
    var messages = socket.bucket('messages');

    messages.subscribe('create', 'update', function(message) {
        // Message created or updated on the server
    });

    messages.subscribe('delete', function(message) {
        // Message deleted on the server
    });

    // Create a new message on the server
    messages.sync('create', { text: 'Hello World' }, function(err, message) {
        // Message saved
    });
</script>
```

Buckets
---

Buckets are stacks of composable middleware functions that are responsible for handling sync requests from the client and responding appropriately.  Each middleware layer is a function that accepts a request and response object (as well as a function that can be called to continue execution down the stack).  A middleware layer will generally either modify the request context and pass control to the next layer or respond to the client with some kind of result or error.

For example, we could add logging middleware to a bucket:

```javascript
var messages = data.bucket('messages');

messages.use(function(req, res, next) {
    console.log(new Date(), req.action, req.data);
    next();
});

messages.use(...);
```

Middleware can be selectively applied to particular actions by specifying them in your calls to `use`.  If a request's action does not match a particular layer then that layer will be skipped in the stack.  For example, we might want to authorize requests on create, update and delete actions:

```javascript
messages.use('create', 'update', 'delete', function(req, res, next) {
    // req.action is one of 'create', 'update' or 'delete'

    req.client.get('access token', function(err, token) {
        if (err) return next(err);

        if (isAuthorized(token)) {
            next();
        } else {
            next(new Error('Unauthorized'));
        }
    });
});

messages.use(function(req, res, next) {
    // req.action could be anything
    // 'create', 'update' or 'delete' is authorized
});
```

Request
---

When clients initiate a sync with the server a request object is created and passed through the middleware stack.  A request object will contain the following properties:

* `action` - the type of sync action being performed (i.e. create, read, update, etc.)
* `data` - any data provided by the client
* `options` - additional options set by the client
* `bucket` - the bucket handling the sync
* `client` - the Socket.IO client that initiated the request

**TODO**: Implement `req.set(key, value)` and `req.get(key)` methods for middleware to store arbitrary data in the request.

Response
---

The response object provides mechanisms for responding to client requests.  It exposes two function `send` and `error` that can be used for returning results or errors back to the client.

Events
---

When a client connects to a particular bucket a `connection` event is emitted on the bucket.  This can be used to perform any initialization, etc.

**TODO**: Implement async handling of connection events (i.e. `var callback = this.async()`).

```javascript
messages.on('connection', function(client) {
    client.set('access token', client.handshake.query.access_token);
});
```

When a response is sucessfully sent back to the client a `sync` event is emitted on the bucket and a sync object is provided.

```javascript
messages.on('sync', function(sync) {
    // Messages bucket handled a sync
});
```

The sync object contains the following properties:

* `client` - the client that initiated the sync
* `bucket` - the bucket that handled the sync
* `action` - the sync action performed
* `result` - the result returned to the client

And provides these methods:

* `notify(emitter, ...)` - emit a sync event on the given emitters
* `stop()` - stop the default notification (syncs are broadcast to all clients by default)

For example:

```javascript
messages.on('sync', function(sync) {
    // Prevent sync event from being broadcast to connected clients
    sync.stop();

    // Notify clients in rooms 'foo' and 'bar'
    if (sync.action !== 'read') {
        sync.notify(sync.client.broadcast.to('foo'), sync.client.broadcast.to('bar'));
    }
});
```

Client
---

The client-side component provides a thin wrapper around Socket.IO for syncing data to the server and listening for sync events triggered by the server.

```javascript
var socket = data(io.connect());
var messages = socket.bucket('messages');
```

Make requests to the server with a bucket's `sync` function:

* `sync(action, [data], [options], callback)` - perform the specified action optionally sending the given data and request options, callback takes an error and a result

```javascript
messages.sync('create', { text: 'Hello World' }, function(err, result) {

});
```

Listen to syncs from the server with a bucket's `subscribe` function:

* `subscribe([action], ..., callback)` - listen for syncs happening on the server optionally passing the actions to which the client should listen, callback accepts a result and action

```javascript
// Listen to all actions
messages.subscribe(function(data, action) {
    console.log(action, data);
});

// Listen to specific actions
messages.subscribe('create', 'update', 'delete', function(data, action) {
    // Action is one of 'create', 'update', or 'delete'
});
```

Install
---

    npm install data.io

Test
---

    npm test

Liscence
---

Copyright (C) 2013 Scott Nelson

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.