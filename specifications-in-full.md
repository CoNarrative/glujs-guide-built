##Specifications in full

We can now return to the specification framework.

As we showed earlier, the specification approach gives the developer and stakeholders a standard way of defining the behavior of a UI that can be directly converted to actionable tests used to both drive development and report results.

In order to run them as tests, we use [Jasmine] (http://pivotal.github.com/jasmine/). A default Jasmine runner is included in the basic project template under the 'spec' folder.

Jasmine itself includes excellent libraries for defining tests. However, it can be a little intimidating figuring out exactly how to structure things - what do you do with the nested describe blocks for instance? GluJS adds a very thin layer of alias functions over Jasmine to strongly guide you down the optimal path for reactive UI testing.

Specifically, GluJS strongly encourages story-based specifications as follows:

###Given

`Given` is used to setup an initial state and should only be used once at the beginning of each spec document. It accepts two arguments: first, a string description of the initial state, and second a nested function that will contain further definitions.

Example (in coffeescript):

```coffeescript
    Given 'the Hello World application on launch', ->
      vm = null      
```

Any 'globals' for this specification should be defined immediately within the nested function so that they are available everywhere. Typically this includes the root vm you are testing, and often a simulated back-end.

###When

`When` blocks are the workhorse of the specification structure and have the same structure as a `Given` block. If a `Given` block starts the overall story, each `When` block advances the story a step or provides an alternate direction. You can tell the difference between the two uses of `When` based on whether you're nesting the function (meaning visually it will also show up indented) or adding it at the same level.

When you nest it within another When statement, we call that adding a *step* to the story. If we want to spell out this story:
 1. Given the Hello World application on launch
 2. When the user toggles their status
    * (check something)
 4. When the user toggles their status (back)
    * (check another thing)

we would do it like so (in coffeescript):

```coffeescript
Given 'the Hello World application on launch', ->  
  When 'the user toggles their status', ->
    When 'the user toggles their status (back)', -> 
```

Since the `->` in coffeescript is just shorthand for a function declaration, the equivalent code in javascript is:

```javascript
Given ('the Hello World application on launch', function(){
    When ('the user toggles their status', function(){
        When ('the user toggles their status(back)', function(){
        });
    });
});
```

The extra syntax in Javascript for defining nested functions is the one and only reason we tend to use coffeescript for our specifications - it keeps them compact and readable and easier to see in plain English.

Sometimes you want to "spec out" additional paths at a certain point in the walk-through. We call these *branches*. Let's say we have two similar stories:
 1. Given the Hello World application on launch
 2. When the user toggles their status
    * (check something)
 4. When the user toggles their status (back)
    * (check another thing)

and the second one:
 1. Given the Hello World application on launch
 2. When the user toggles their status
    * (check something)
 4. When the user executes the "Do" command
    * (check another thing)

We instantly notice that the first 2 steps are the same, and only the last step is different. Once you add a full set of descriptive user stories, you're going to find that every specification story overlaps with at least one other, leading to a tremendous amount of duplication and also making it difficult to see what is going on.

While we could keep these two stories separate, GluJS with Jasmine gives us the ability to add the second story as a *branch* of the first. We do this in code by simply adding another block *after* the first as follows:

```coffeescript
Given 'the Hello World application on launch', ->  
  When 'the user toggles their status', ->
    When 'the user toggles their status (back)', ->
      #stuff that deals with toggling the status back after the first toggle
    When 'the user executes the "Do" command', ->
      #stuff that deals with executing the "Do" command after the first toggle
```

Another way of looking at specifications is that they represent the "user acceptance testing" that you would give to a manual tester after the application was built. Whatever you would have to spell out to a manual tester should be the steps and branches you put into a `When` block.

Notice that 'Given' and 'When' blocks are simply structure and do *absolutely nothing* on their own. To actually implement the original state or to make changes, we use the `Meaning` block.

###Meaning

`Meaning` is the block that actually advances the simulated state of the application. The goal is to let the view model "do its thing" entirely ignorant of whether it is running in test harness or as a live applicaiton. That means it has to deal with everything that could happen within the client. These break down into four basic categories:
 *  *Simulation setup* - Before anything can run, the simulation must be placed into a defined test harness.
 *  *The user takes an action* - The user of the application at this point clicks, drags, or othewise interacts with the application, such as:
    * Clicks on a button (to execute a command)
    * Selects a row or item from a grid/list
    * Updates text in a text field or area
    * Changes the focus of a list of things
 *  *An asynchronous timed background process executes* - The application might have a periodic refresh, or a one-time delayed action that was triggered by an earlier user command.
 *  *An asynchronous Ajax call returns a response* - The application receives a response from the server. This could be a success, a failure, or even a timeout.

GluJS combined with Jasmine provides complete simulation coverage for all four categories of action:

####Simulation setup

Usually you are going to define and set up your root view model using `glu.model` since that function gives you root view model without creating a view:
```coffeescript
    Given 'the Hello World application on launch', ->
      vm = null
      Meaning -> vm = glu.model 'helloworld.main'
```

Most of the time, you are also going to be setting up a simulated "back end" so that you can track and respond to Ajax calls. The application will make its calls as usual, only glu will now intercept and process the Ajax calls locally without a server having to be involved. Of course you'd need a reasonable way to quickly declare Ajax "routes" (url matching patterns) and serve up data. This is exactly what GluJS provides and is described under the 'data' and 'ajax' section below.

Lastly, you may want to advance the application to a beginning state by initializing the view model and answering the Ajax calls it makes on initialization. For instance, if the Hello World app needs to get a roster of other people who are also logged in, you might want to advance to that point:

```coffeescript
    Given 'the Hello World application on launch', ->
      vm = null, backend
      Meaning -> 
        vm = glu.model 'helloworld.main'
        backend = helloworld.createMockBackend() //a function we define ourselves elsewhere
        vm.init() //internally makes a call to the 'roster' url
        backend.respondTo 'roster'
```

####User actions

User actions are the simplest because they emerge naturally out of what you've defined on the view model. We earily introduced the concept of "interesting" interactions. Another way of looking at that is to ask if it is something you would want to introduce as a step in a manual test. If so, it is "interesting" and belongs in the view model as a public command or property.

Here's a concrete example using the helloworld example:

```coffeescript
Given 'the Hello World application on launch', ->
  #...Given block stuff...
    When 'the user toggles their status', ->
      Meaning -> vm.set 'isLeaving', true
```

Since we know that all bindings are "simple" (have no logic) we know that toggling the button is simply going to set isLeaving alternately to false and true. Rather than try and locate the button on the DOM or within the ExtJS component tree, we simply do the same thing and set the property.

If it is a user command, we just invoke the function (just as the button would).

```coffeescript
Given 'the Hello World application on launch', ->
  #...Given block stuff...
    When 'the user executes the "do it" command', ->
      Meaning -> vm.doIt()
```

If it is a text or value field of some sort, we just set the value:

```coffeescript
Given 'the Hello World application on launch', ->
  #...Given block stuff...
    When 'the user changes the personalized welcome to "foo"', ->
      Meaning -> vm.set 'welcomeMessage', 'foo'
```

If it is selecting a row in a grid, we just find the one we want from the store and set it:

```coffeescript
Given 'the Hello World application on launch', ->
  #...Given block stuff...
    When 'the user selects a person in the roster', ->
      Meaning ->
        vm.set 'personSelection', vm.personList.getAt(0) //set selection to first item in list
```

If it is responding to a dialog there's a special method off of the confirm method called `respondWith`:
```coffeescript
    When 'the user confirms the delete action', ->
      Meaning -> vm.confirm.respondWith('yes')
```

Note that we should never set a formula or invoke anything directly that isn't bound to a control - the point of GluJS story-based testing is that you emulate what the *user does* or what the *app does in response or on a timer*, so that the tests are as non-brittle, full-coverage, and as easy-to-follow as possible. Let the application handle its own internals.

#####Including the view in testing (sidebar)

We have also experimented with "view-inclusive" testing, but that makes things more difficult.

We know that we don't want to render the DOM during normal cycles(too slow and difficult to control), but that still leaves us two options. First, we could just let GluJS bind the ExtJS object tree but not render it. That's simple (coffeescript):

```coffeescript
    var vm = glu.model ('helloworld.main')
    var view = glu.view vm
```

The `glu.view` function delivers a fully bound view without adding it to any container that would trigger DOM rendering.

That would allow us to validate the bindings (in addition to the behavior) by driving changes indirectly through the controls instead of through the view model. Of course, the challenge becomes then locating the view components without making the tests "brittle" (breaking tests because of shifting around a view is hugely draining). While testing, there is in fact basic support for quickly finding view components based on their bindings to the view model. We provide a "control lookup" for every bound property or command on the view model so that you can find what is bound to it without doing component or DOM queries (this is only available in test mode).

Unfortunately, many ExtJS controls assume that they are rendered before you can begin poking around with them. "Headless" adapters for ExtJS that would correct that behavior are possible (we've built some) but too much effort at the moment for little return. Since bad bindings typically throw errors in any case, and since you can validate your bindings with existing support, there is not much to be gained by driving the tests through the controls themselves.

The second remaining option is simulating the DOM within a javascript engine like Node.js. While entirely possible, that becomes a more complicated process all around and defeats the GluJS simplicity of a single developer being able to immediately produce and run testable code without additional installs or environment setups.

Since the views are behaviorless within GluJS and everything "interesting" (custom) goes into the view model, testing the view models is a simple and sufficient way to provide excellent coverage. 

If you're still interested in "headless ExtJS controls" and perhaps a quick "binding validation check", let us know since we're considering addressing those points in the future.

####Background jobs

Jasmine already has excellent support for manually controlling the clock to handle background tasks. Here's an example of advancing the clock in the hello world application:

```coffeescript
  Given 'the Hello World application on launch', ->
    vm = null, backend
    Meaning ->
      vm = glu.model 'helloworld.main'
      vm.init() #starts a timer such that if you don't talk to someone within 2 minutes it ask you what is wrong
      jasmine.Clock.useMock() #tell jasmine to simulate the clock
      #...validate that message should be still Hello World!
    When 'it is over 2 minutes later'->
      jasmine.Clock.tick 2*60*1000 + 1000 #advance the clock 2 minutes 1 second
      #...validate message should now be What is wrong?
```

####Ajax responses

GluJS lets you simulate Ajax responses using the GluJS [AJAX simulator](simulation-framework.md#ajax-simulator). You define the routes during setup and turn on the framework for *capture*. As the calls are made, instead of being delivered to the browser for an actual AJAX call, they are stored up in each captured route by name.

You will typically have a different route for each type of service call and name them accordingly. That way you can manage the responses separately and logically.

For example, let's say that the 'add to group' operation triggers a single 'add to group' service which when finished triggers a grid refresh. Simulating that might look like the following:

```coffeescript
  #...Given setup here...
  When 'the user executes the "add to group" command', ->
  Meaning -> vm.addToGroup()
    When 'the back-end returns the 'add group' response successfully', ->
      #technically don't need a status as that defaults to 200
      Meaning -> backend.respondTo 'addGroup', {status:200, responseText : "{}"}
      When 'the back-end returns the people roster', ->
        #Leaving off the response object lets a centrally defined fake service handle it, and so can keep state within a story
        Meaning -> backend.respondTo 'persons'
```
Remember, we don't trigger the calls themselves -- the view model does that as it goes about its normal business. We just provide the simulated response from the backend in order to drive the story forward.

###ShouldHave

At this point, we are successfully *simulating* the application (which is nice) but not specifying what *should happen*, meaning we are also not *testing* it. That's where the `ShouldHave` blocks come in.

An example:

```coffeescript
Given 'the Hello World application on launch', ->
  vm = null
    Meaning -> vm = glu.model 'helloworld.main'
    ShouldHave 'set the message to "Hello World!"', -> (expect vm.shoutOut).toBe 'Hello World!'
    When 'the user toggles their status', ->
    Meaning -> vm.set 'isLeaving', true
       ShouldHave 'set the message to "Goodbye World!"', -> (expect vm.shoutOut).toBe 'Goodbye World!'    
```

`ShouldHave` blocks assert expectations about the result of the store step - what *should have* happened or what the UI now *should have*. It is an alias on the Jasmine `expect` function, though we prefer to use `ShouldHave` as it keeps writing all of our expectation sentences consistently. The ShouldHave function receives two arguments - the expectation in plain English, and then a function that contains one or more *expectations*, also known as *assertions*.

For the expectations, we simply use the excellent out-of-the-box Jasmine matchers. However, there are a number of GluJS helper functions:

####getRequestsFor

One of the most commonly used helper functions when simulating Ajax is `backend.getRequestsFor()`. This lets you read and set expectations on interactions with the back-end.

For example, let's flesh out the example given above with the actual expectations. We're not only going to respond to the calls, we're going to assert that they were made in the first place:

```coffeescript
  #...Given setup here...
  When 'the user executes the "add to group" command', ->
  Meaning -> vm.addToGroup()
    ShouldHave 'made a call to the 'add group' backend service', -> (expect backend.getResponsesFor('addGroup').length).toBe(1)
    When 'the back-end returns the 'add group' response successfully', ->
      Meaning -> backend.respondTo 'addGroup'
      ShouldHave 'made a call to the 'persons' backend service', -> (expect backend.getResponsesFor('persons').length).toBe(1)
      When 'the back-end returns the people roster', ->
        Meaning -> backend.respondTo 'persons'
```

Technically you don't have to do this - the later `respondTo(route)` call will throw an exception if there was no valid request. This pattern simply spells it out in greater clarity.


*Copyright 2012 Mike Gai. All rights reserved.*