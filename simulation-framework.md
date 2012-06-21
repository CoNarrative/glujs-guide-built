##Simulation framework

In this section, we go deeper into the setup and use of the GluJS simulation framework.

###Ajax simulator

Use of the Ajax simulator involves a few simple steps:

 * Set up a backend with routes

 * Turn on capturing of Ajax calls

 * Returning response objects as needed

####Routes

To set up routes on the back-end is a simple matter of using the `glu.test.createBackend` function:

```javascript
glu.ns('assets').createMockBackend = function(liveMode) {
    var assets, backend;
    assets = glu.test.createTable(assets.models.asset, 11);
    backend = glu.test.createBackend({
        defaultRoot: '/json/',
        fallbackToAjax: liveMode,
        autoRespond: liveMode,
        routes:  {
            'removeAssets' : {
                url: 'assets/action/remove',
                handle: function(req) {
                    return assets.remove(req.params.ids);
                }
            },
            'requestVerification' : {
                url: 'assets/action/requestVerification',
                handle: function(req) {
                    return assets.update(req.params.ids, {
                      status: 'verifying'
                    });
                }
            },
            'assetSave' : {
                url: 'assets/:id/action/save',
                handle: function(req) {
                    return assets.replace(req.params.id, req.jsonData);
                }
             },
            'assets' : {
                url: 'assets',
                handle: function(req) {
                    return assets.list(req.params);
                }
            },
            'asset' : {
                url: 'assets/:id',
                handle: function(req) {
                    return assets.get(req.params.id);
                }
            }
        }
    });
    backend.capture();
}
```

The `routes` property is the most important. It consists of one or more named 'back-end service' definitions (or *routes*) that mimic a back-end web interface.

The URL for each route is a simple pattern containing url (REST) parameters to capture. These captured parameters show up in the `params` collection, as well as any key-value pairs from the query-string (make sure there is no overlap).

After the call to `.capture`, whenever a call is made to the view provider's (e.g. ExtJS) Ajax library it will first attempt a match against one of the provided routes. If there is a match, it will store the request in the route's request queue.

If there isn't a match, it will do one of two things based on the `fallbackToAjax` setting. While testing, ordinarily you want that to be `false` so that any uncaptured routes throw an error - you shouldn't be testing against routes that you aren't emulating. The only time to use `fallbackToAjax` is when you are embedding the application into a larger application as a 'live demo'.

The call to an Ajax request is asynchronous (as expected) and so nothing else happens at this time other than adding it to the queue or throwing a "route not found" exception.

####Response objects

You control when the simulate Ajax call returns by the `respondTo` method. You can provide a custom one-time response when you call the function; this is especially useful when simulating errors.

For example (in CoffeeScript):

```CoffeeScript
  #...Given setup here...
  When 'the user executes the "add to group" command', ->
  Meaning -> vm.addToGroup()
    When 'the back-end add group service call fails unexpectedly', ->
      Meaning -> backend.respondTo 'addGroup', {status:500, responseText : "Internal Server Error"}
```

In addition to supplying the HTTP `status` code (default 200) and `responseText`, you can also return HTTP headers in a `headers` object as needed:
```CoffeeScript
      Meaning -> backend.respondTo 'addGroup', {
          responseText : '{id:1}',
          headers : {
            'CONTENT-TYPE' : 'application/json' #the default actually
        }
```

As a convenience, you can also return a `responseObj` instead of a `responseText`. This will (as you might guess), simply serialize the provided object into the responseText:

```CoffeeScript
      Meaning -> backend.respondTo 'addGroup', {
          responseObj : {id:1}
        }
```

And because more often than not you will be returning a successful response instead of a failure, there's a final shortcut in which you simply provide the responseObj itself (equivalent to the previous example):

```CoffeeScript
      Meaning -> backend.respondTo 'addGroup', {id:1}
```

Race conditions can be more easily caught because now you can spell out the exact order of the Ajax return calls. If you are having trouble isolating a race condition, simply have the Ajax calls return in various orders until you have isolated it. Now that can become a permanent test case to not only help you fix and test, but also prevent a regression in the future.

####Live demo / user training mode

The Ajax simulator is *not just for testing*. It also lets you quickly stand up your client application for interactive demonstration and testing purposes. Simply set `autoRespond` to true in the back end and instead of queuing requests up, they will automatically respond asynchronously from the in-memory backend. If the application is a module in a larger application that may already be hitting a live backend, you can still use the simulation framework: just set `fallbackToAjax` to true and routes that you are not capturing will hit the actual server as usual.

The ability to 'live demo' a client UI iteration is one of the more powerful features of GluJS and can play a key part in a fast, Agile development workflow. It also can be a very useful way to conduct user training when constructing a fully separate training back-end is too costly to manage.

###Data simulator

Faking out the back-end services so that they actually, pretty-much *work* is extremely valuable for two reasons:

 1. It lets you make your stories correspond to actual cases which involve the back-end, instead of being artificially chopped up to avoid that
 2. It provides a way to quickly iterate and converge on a client UI without waiting for a back-end service to be written (often by another team)
 3. By enabling a 'live demo' mode, it lets stakeholders quickly communicate likes and dislikes on the developing interface very early in the design until after everything is set in stone
 4. Since stakeholders often are "touch and see" oriented instead of dealing with abstract development arcana, this process can actually be used to flesh out service contracts and patterns in advance of their implementation.

Since back-ends revolve around persistent data in databases, GluJS will not only have to be able emulate AJAX, but also provide a simple way to fake persistent storage. And so it does.

In the example under the routes, there is a line in which we set up an in-memory database:

```javascript
    assets = glu.test.createTable(assets.models.asset, 11);
```

That will set up a database table based on the `fields` definition in the asset model (which can be a GluJS view model or an ExtJS Model construct0). The second argument (11) tells it to populate with 11 faked rows using the GluJS fake data helpers. Alternately, you can provide an array of objects and those will be read into the database upon initialization.

The returned table object supports the usual set of CRUD operations:

 * `create` - create a new row

 * `list` - return a paged, sorted, filtered list

 * `get` - return a single item by key

 * `update` - update the fields of a single item

 * `replace` - complete replace the row of a single item (equivalent to a remove and a create with the same key)

 * `remove` - remove an item from the list based on the provided key

All of these accept standard params (such as `start`, `limit`, etc.) and return a "standard response" (list returns a paged response for instance), so if you want to simply let them handle the received parameters and return a standard result for your REST calls, you can do that:

```javascript
            'assets' : {
                url: 'assets',
                handle: function(req) {
                    return assets.list(req.params);
                }
            },
```

The example above will respond to sorting, paging, and filtering requests as generated by the ExtJS store and will return an object that looks like the following:

```javascript
    {
        "total":11,
        "rows":[{id:1,name:'Katie'},{id:2,name:'Mike'},...etc....]
    }
```

The database is one-table at a time for now. Anything more complex (involving joins, etc.) you can implement manually as needed.


####Fake data

You can lean on the fake data API which will provide fake data based on an analysis of the names of the fields, their data types, and their constraints. Often you won't be using it directly but simply relying on the GluJS data layer to create it for you when you make a sample table. See the API for more details.

Building out the ease and sophistication of the ajax and data frameworks is one of our top priorities on the GluJS roadmap and we are greatly interested in your feedback.


*Copyright 2012 Mike Gai. All rights reserved.*