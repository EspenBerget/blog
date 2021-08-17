---
title: "Trying Deno and Building a Server API"
date: 2021-08-17T17:44:46+02:00
draft: false
---

I've looked at Deno a couple of times, but never tried it. About a week ago I decided that it was time. Deno have some nice builtin features that you have to use third party libraries to get with node. 

I decided to see what it was like to make something using Deno.

## Deno
What deno will give me is a easy way to manage my dependencies, and also running and watching the app effortlessly. 
It also give me stricter control of the resources used by my app, for example networking or reading/writing to files. 

### Imports
So in deno you can import a resource directly from the web. This is something Go users will be familiar with, but we also do it 
in JavaScript, just not in node. In HTML on the other hand we can import resources from other sites with the script tag. Deno gives us this directly in the `import` statement. We will use this to import two libraries that we need to make our server.
They are:
```javascript
import { DB } from "https://deno.land/x/sqlite@v3.0.0/mod.ts";
import { serve } from "https://deno.land/std@0.103.0/http/server.ts";
```

> ##### These imports bind our version to a given release. This is good practice as without explicitly stating package version we can end up with a new version of http server in the middle of not only production, but also development. 


With these two imports I have declared all my dependencies for this app, no package.json needed. 

## The Database
The database is dead simple. It only holds a single table called events. It is used to track all types of events given only a
description of the event. Its not ment to be practical, only to serve the role of a issue that needs the typical HTTP request methods
to be implemented. We can make an event, delete it, patch it, or query it. The table looks like:
```sql
    CREATE TABLE IF NOT EXISTS events (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        desc TEXT,                                       -- 1.
        at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
    )
```
1. desc is short for description.

Using the `DB` import from above. We make a new instance with
```javascript
const db = new DB('events.db');
```
Now we have a sqlite file called `events.db` and a way to interface with it using the `db` object. The function `db.query` will
be heavily relied on, as it runs all sql queries, be it making a table, updating or removing.

##### **`db.js`**
```javascript
import { DB } from "https://deno.land/x/sqlite@v3.0.0/mod.ts";

// Open database
const db = new DB("events.db");

// Setup table 
db.query(`
    CREATE TABLE IF NOT EXISTS events (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        desc TEXT,
        at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
    )
`);

// API
export function addEvent(text) {
    db.query("INSERT INTO events (desc) VALUES (?)", [text]);
}

export function removeEvent(id) {
    db.query("DELETE FROM events WHERE (id == ?)", [id]);
}

export function updateEvent(id, text) {
    db.query("UPDATE events SET desc = ? WHERE (id == ?)", [text, id]);
}

export function getEvent(id) {
    return db.query("SELECT * FROM events WHERE (id == ?)", [id]);
}

export function getAllEvents() {
    return db.query("SELECT * FROM events");
}

export function closeDB() {
    db.close();
}
```

If you know sql, then the only thing that might be foreign her is the `?` in the queries.
It is a placeholder for the values passed in an array to the second argument of `db.query`. 
It's important to do it like this as it hinders sql-injections.


## My Server API
Like express, I wanted the server to expose different request methods as functions.
So if you need to make a get request handler you would write:
```javascript
server.get(route, handler);
```
The `route` is a path in the url and the `handler` is a function that expects a `ServerRequest` imported from the deno http server library.

We might also want to add middleware, so our function will look like:
```javascript
server.get(route, ...handlers) { ... }
```
All the handels run before the last are middleware. They can for instance parse the query part of a url.

### Server class
The Deno http server gives us the main listening and request functionality, and the `Server` class will give a simple way to organize it. This is how it looks:
```javascript
export class Server {
    constructor(port, loggingOn = false) {
        this.port = port;
        this.server = serve({ port });

        // route -> (method -> handler)
        this.routes = new Map();
    }
    ... 
}
```
The main things to note is the call to `serve` which gives us a server on the port given by the user, and the `this.routes` variable, which holdes the routes and methods per route. 
We use this when adding routes and when matching a request.

### Handlers
First up we should make the handle method. This thing will be responsible for assigning the handlers to a route and request method.
```javascript
handle(route, handles, method) {
     let methods = this.routes.get(route);   // 1.

     if (methods === undefined) {
         methods = new Map();                // 2.
         this.routes.set(route, methods);    // 2.    
     }

     methods.set(method, async req => {      // 3.
         for (const h of handles) {
             await h(req);                   // 3.
         }
     });
 }
```

1. We see if there already is a method map in this route.
2. If not we make a new one, and assign it.
3. Here we make our own async handler, and assign it to the request method. This will run all the handlers in sequence, so that our middleware can do its thing before we continue. This is why we call await inside the for loop.

With this in place we can make a method in `Server` for each request method.
```javascript
 get(route, ...handles) {
     this.handle(route, handles, "GET");
 }

 post(route, ...handles) {
     this.handle(route, handles, "POST");
 }
 
 patch(route, ...handles) {
     this.handle(route, handles, "PATCH");
 }

 delete(route, ...handles) {
     this.handle(route, handles, "DELETE");
 }
```

### Running The Server
The server will need to be started in a way. For this we will use a async method called listen.
```javascript
async listen() {
    console.log(`Starting server on localhost:${this.port}`); 

    for await (const req of this.server) {  // 1.
        this.run(req);                      // 2.
    }
}
```

1. This loop runs forever waiting for requests from the server.
2. When we get a request we will run it.

Now we need to make the run method.
```javascript
run(req) {
    // this localhost stuff is a hack but it will work fine, and 
    // give us the needed functionality from the URL class.
    let url = new URL("https://localhost" + req.url);
    let methods = this.routes.get(url.pathname);            // 1.
    if (methods === undefined) {
        req.respond({ status: 404, body: "Not Found" });    // 3.
        return;
    }
    
    let handle = methods.get(req.method);                   // 2.
    if (handle === undefined) {
        req.respond({ status: 404, body: "Not Found" });    // 3.
        return;
    }

    handle(req);                                            // 4.
}
```

1. Check the routes for a match on the url pathname, not the whole url.
2. Then see if the route has the correct request method.
3. If not found return a 404 status.
4. If all goes well, run the handler by passing it the request object.

> ##### I use the URL class, but it expects a complete url, and `req.url` is just the path part. So to fix this I append http://localhost. This does not change anything for our use, but it is not pretty. This in necessary because an url might also have a query like '?id=3' and this will be part of `req.url`, but not url.pathname. 


### Utility
There is one last function that I want to expose. It allows us to return JSON in a simple manner.
```javascript
export function respondWithJSON(req, obj) {
    req.respond({body: JSON.stringify(obj)});
}
```
Now when a handler want to return JSON it simply calls this function with the request and object to return.

## Using The Server
Now that the API is ready we should try it. In a separate file, that I named `index.js`, we can import the server and initialize it.
```javascript
import { Server, respondWithJSON } from './server.js';

const server = new Server(8001);
```
Now lets make a simple get handler and run the server.
```javascript
server.get('/', req => {
    req.respond({body: "hello world"});
});

await server.listen();
```
If we run 
```bash
$ deno run --allow-net index.js
```
and in another terminal window run
```bash
$ curl http://localhost:8001/
hello world
```
And it works!

### Building a Proper Web API
With the server and database APIs ready, its only proper to expose it to the world. We need a web API. First we will add two import statements to `index.js`.
```javascript
import {addEvent, removeEvent, updateEvent, getEvent, getAllEvents, closeDB} from './db.js';
import {getJSON, checkJSON, getQuery} from './middleware.js';
```

The database import should be familiar, but the middleware is new. It is not crucial to know how it does what it does, but I will show it now for the curious, and the ones how are following along. For the rest just know that, `getJSON` parses JSON from the request body, and exposes it as `req.json`. `checkJSON` makes sure it is correctly formatted, and `getQuery` parser the query part of an url and exposes it as `req.query`.

##### **`middleware.js`** 
```javascript
// JSON
export async function getJSON(req) {
    let buf = await Deno.readAll(req.body);
    req.json = JSON.parse(new TextDecoder().decode(buf));
    return Promise.resolve("done");
}

export function checkJSON(correct) {
    return req => {
        req.jsonErrors = "";
        if (req.json === undefined) {
            req.jsonErrors = "no JSON found";
        } else {
            for (const k in correct) {
                if (req.json[k] === undefined) {
                    req.jsonErrors += ` ! missing ${k} of ${correct[k]}`;
                } else if (typeof req.json[k] !== correct[k]) {
                    req.jsonErrors += ` ! wrong type of ${k}, is ${typeof req.json[k]} but should be ${correct[k]}`;
                }
            }
        } 
    }
}

// QUERY
export function getQuery(req) {
    req.query = {};

    // this localhost stuff is a hack but it will work fine, and give us the 
    // needed functionality from the URL class.
    let url = new URL("https://localhost" + req.url);

    for (const k of url.searchParams.keys()) {
        req.query[k] = url.searchParams.get(k);
    }
}
```

Now for the web API. We need to get back to `index.js` and make the handlers. First lets look at getting all events.

```javascript
server.get('/events', req => {
    const events = getAllEvents();
    respondWithJSON(req, events);
});
```

This one listens on the path localhost:8001/events for GET requests.
When one comes it queries the database for all events and sends them to the client with JSON encoding.

That was simple enough, lets see if we can query a single event.
```javascript
server.get('/event', 
    getQuery,
    req => {
        if (req.query.id !== undefined) {
            const event = getEvent(req.query.id);
            respondWithJSON(req, {event});
        } else {
            respondWithJSON(req, {status: "query param 'id' is missing"});
        }
    }
);
```
We simply place the `getQuery` middleware function ahead of our handler and check if `id` was part of the query. 
Deleting is pretty much the same.
```javascript
server.delete('/event', 
    getQuery,
    req => {
       if (req.query.id !== undefined) {
           removeEvent(req.query.id);
           respondWithJSON(req, {status: "deleted"});
       } else {
           respondWithJSON(req, {status: "query param 'id' is missing"});
       }
    }
);
```

More interesting is adding a new event. This will utilize the post method.
```javascript
server.post('/event', 
    getJSON,                                                
    checkJSON({desc: "string"})                             
    req => {
        if (req.jsonErrors === "") {
            addEvent(req.json.desc);
            respondWithJSON(req, {status: "ok"});
        } else {
            respondWithJSON(req, {status: req.jsonErrors});
        }
    }
);
```
`getJSON` does what `getQuery` did in the previous handlers. It adds the result in the variable `req.json`. 
The other function is `checkJSON` it take a object describing the expected json structure, and returns a middleware that controls that the json is correctly structured. Any mistake is added to `req.jsonErrors`. Should it be empty, the json is correct. This particular instance says we expect a key called `desc` of type `string`.

Now as a final test, we put everything together in the patch handler.
```javascript
server.patch('/event', 
    getQuery,
    getJSON,
    checkJSON({desc: "string"}),
    req => {
        if (req.query.id !== undefined) {
            if (req.jsonErrors === "") {
                updateEvent(req.query.id, req.json.desc);
                respondWithJSON(req, {status: "patched"});
            } else {
                respondWithJSON(req, {status: req.jsonErrors});
            }
        } else {
            respondWithJSON(req, {status: "query param 'id' is missing"});
        }
    }
);
```

There is nothing new here, just that all these things can be combined, and reused.

### Trying It Out
First up we need to start the server.
```bash
$ deno --allow-net --allow-read --allow-write index.js
Starting server on localhost:8001
```

Next up we open another terminal and try to post something. 
```bash
$ curl -X POST localhost:8001/event \
       -H "Content-Type: application/json" \
       -d '{"desc": "hello world"}'
{"status":"ok"}
```
Neat! Now lets see our result.
```bash
$ curl localhost:8001/events
[[1,"hello world","2021-08-16 15:43:37"]]
```
This returns an array of events. Each event is itself an array of id, desc, and timestamp, functioning as a tuple. 

Now lets patch our event.
```bash
$ curl -X PATCH 'localhost:8001/event?id=1' \
       -H 'Content-Type: application/json' \
       -d '{"desc": "bye world"}'
{"status":"patched"}
```

Now lets check it again, but this time only request this event.
```bash
$ curl 'localhost:8001/event?id=1' 
[[1,"bye world","2021-08-16 15:43:37"]]
```
This returs an array of events, but we know that at max there should only be one element. 
So really this array is more like Haskell's `Maybe` type. Either there is nothing in the array, or ther is just one event there.

Finally lets remove the event.
```bash
$ curl -X DELETE 'localhost:8001/event?id=1'
{"status":"deleted"}
```
Checking that it was removed gives us
```bash
$ curl localhost:8001/events
[]
```
No more events.

## Conclusion
This project, as a test for Deno can at best claim what Deno can do for a hobbyist. And as a hobbyist I am very satisfied. The functionality I needed was there, and easy to import. The lack of any initialization made the "idea to product" development process really effortless. 

I ended up with having to explain more about the server aspect of this project than I had hoped for, but hope that the readers will find those parts interesting as well, and that on a surface level deno seems like a proper replacement for node. Its even made by the same guy, the name is just an anagram for node. 

Next time I visit deno I will try to look at how having the web API on the backend can be useful. Deno leverages things like web workers, and the fetch API, and I want wait to make something from it.


This repository have a multi purpose. My next post will be how to set up the environment (Deno and Svelte), using nix. 


### P.S
This repository is available on [my github][rep], where there is also a frontend in Svelte do show that the backend works. 
There are some more functionality not addressed here like CORS and OPTIONS request handlers. 

[rep]: https://github.com/EspenBerget/my-server-api
