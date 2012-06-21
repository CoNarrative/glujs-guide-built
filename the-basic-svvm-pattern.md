##The Basic SVVM Pattern
GluJS is a refined variant of the MVVM (Model-View-View Model) pattern that we call SVVM. That's short for Specification-View-View Model.

The switch to SVVM underscores two points. First it underscores one of the main points of GluJS: making it dead simple to drop right into test-first development based on user stories or specifications. In fact, we recommend (and make it simple) to always start with a specification.

The second is that the architecture is simpler than it looks. Essentially, there are

 * specs -        descriptions of what the app is supposed to do that are also fully functional tests

 * view models -  descriptions of screen states and rules that include data

 * views -        declarative JSON that describes the component tree (similar to MXML or XAML from Flex and WPF/Silverlight)

That's really it. By simplifying the architecture, we make it a very direct path to true test-first development.

###What is a specification?

Let's start with an example. Imagine a "Hello World" application. Except since we're building reactive applications, imagine it has some minimum behavior as follows:

 * There's a "message" label that contains either "Hello World!" or "Goodbye World!"

 * It starts out with "Hello World"

 * There's a toggle button that tracks whether you are coming or going.

 * When you click it, it flips over to "Goodbye World!"

To formalize this a little, we'll put this in the shape of a user story. We are going to use the "Given, When, Should Have" format. In other words, almost any user story can be reduced to a few basic parts:

 * *Given* - the initial state you're going to be describing

 * *When*  - the change you are going to introduce (usually a user interaction, scheduled event, or feedback from the back-end)

 * *Should have* - the "then" of what happens. In other words, it "should have" done one or more things in response, like change parts of the screen or talked to the back-end.

With that in mind, the Hello World "story" specification might look like this:

    Given the Hello World application on launch
      Should have set the message to 'Hello World!'
      When the user toggles their 'is user leaving' status
        Should have set the message to 'Goodbye World!'

Indenting just means the next "step" in the story. Why not leave it flat instead of indenting? Because indenting lets us combine *multiple related stories* into a single clear, readable spot. Consider if there was another option such as "change goodbye greeting" that you could change before flipping the toggle. Instead of writing an entirely new story, you could just append it to the existing one:

    Given the Hello World application on launch
      Should have set the message to 'Hello World!'
      When the user toggles their 'is user leaving' status
        Should have set the message to 'Goodbye World!'
      When the user sets the goodbye message to 'Ciao!'
        Should have left the current message alone
        When the user toggles their 'is user leaving' status
          Should have set the message to 'Ciao!'

In other words, we've just added a new "branch" to the set of stories--a different path the user could take at that point. We added the branch at the beginning, but it is especially powerful when you realize you can add it at any point. That lets you combine all of the "setup steps" for two stories that start out at the same spot (for instance deep in a set of screens and interactions) but only diverge in the last steps. More on that later.

We recommend capturing your user stories as much as possible in this format. The compelling reason is that if the stakeholder, business manager, PM, etc. can spell it out at this level, you can eliminate a world of development pain. Why? Because this level of specification can be directly translated into actionable tests to guide the developer to get it right the first time.

####The specification in code

Here is that same (first) example given in CoffeeScript:

```CoffeeScript
Given 'the Hello World application on launch', ->
  vm = null
  Meaning -> vm = glu.model 'helloworld.main'
  ShouldHave 'set the message to "Hello World!"', -> (expect vm.shoutOut).toBe 'Hello World!'
  When 'the user toggles their status', ->
    Meaning -> vm.set 'isLeaving', true
    ShouldHave 'set the message to "Goodbye World!"', -> (expect vm.shoutOut).toBe 'Goodbye World!'
```

We use CoffeeScript (for the specifications only) because it makes them so short and expressive that they match up very closely with the plain English specification provided by the analyst. Here's the same thing in javascript if you prefer that:
```javascript
Given ('the Hello World application on launch', function() {
    var vm;
    Meaning (function(){
        vm = glu.model ('helloworld.main');
    });
    ShouldHave ('set the message to "Hello World!"', function() {
        expect (vm.shoutOut).toBe ('Hello World!);
        When ('the user toggles their status', function() {
            Meaning (function(){
                vm.set('isLeaving',true);
            });
            ShouldHave ('set the message to "Goodbye World!"', function(){
                expect (vm.shoutOut).toBe ('Goodbye World!');
            });
        });

    });
});
```

Either way works. Don't worry about what's going on in the details - we'll return to that in a moment. Just note how similar the developer specification is to the plain English specification. The "miscommunication" gap has been reduced to a minimum and the developer can start coding exactly "to spec".

###The view model

You've seen an overview of a specification. What exactly does the developer code to make this come to life? The first thing the developer needs to do is model the application behavior.

A view model is a "state machine" that tracks anything "interesting" going on in your application. What we mean by interesting is simply any screen behavior that includes one of the following:

 * app-specific logic that is not encapsulated in a control and you'd have to write some sort of handler for (especially when one control influences another). In short, if you would ordinarily go about it by adding a handler to a control or calling a method on a control, you should put that in the view model.

 * the state of any forms / data made visible in a screen

 * validations (really another form of app-specific logic)

 * calculations (the same again)

 * Ajax calls to a "back-end"

It does *not* include "uninteresting" things that are part of standard out-of-the-box control behavior. For instance, whether a combo-box is expanded or not is usually based entirely on existing standard conventions and handled internally by the control.

The moment that behavior has additional app-specific logic, it becomes part of the view model. For instance, if the expanded/collapsed state of the combo box is supposed to be auto-expanded whenever a value of 'Other' is input into another combo-box, then that logic needs to be put into the view model as it is not provided "out of the box".

Here's an example of a simple view model that fulfills the behavior in the spec above:

```javascript
glu.defModel('helloworld.main',{
    arriving : true,
    message$: function() {
        return this.arriving ? "Hello World!" : "Goodbye World!"
    }
});
```

The view model has a single property and a single formula. A property is simply what you'd expect - a value that you change by calling `.set('arriving')` on the view model. It raises event notifications whenever it changes value.

A formula is a special sort of property that is not changed by calling `.set`, but instead re-calculates whenever referenced values change, much like a spreadsheet cell that points to other cells. In the case above, the value of `message` will be changed if and only if the value of 'arriving' changes. A property is marked as a formula simply by appending a `$` to it and then providing a function that returns a value. The GluJS framework will analyze the function and "wire it up" accordingly.

This view model above will start with an `arriving` of true and a `message` of 'Hello World'. If you change the value of arriving by calling `.set('arriving', false)`, `message` will change to 'Goodbye World!'.

In fact, this is the exact behavior we described in our specification. If we run the specification in the Jasmine spec runner, it will return all green, meaning tests passed.

The view model is *entirely independent* of any view. You can validate your "interesting behavior" before you even can see it in operation. The view, because it has no logic, simply attaches and follows the view model like the "skin" of a 3d model over the skeleton.

We will cover the details later, but this gives you a good idea of what a basic view model contains.

###The view and binding
The view model can be run without any view. It is the actual application that you are designing, abstracted from its representation. Of course, it is of little value without the visual "skin" to make what is going on available to the user. This we call the "view".

One of the main goals of GluJS is helping the underlying simplicity of the application emerge. Here is an example of an ExtJS view for the Hello World application:

```javascript
glu.defView('helloworld.main',{
    title: '@{message}',
    tbar : [{
        text : '~~arrivalStatus~~',
        pressed : '@{arriving}'
    }]
});
```

Views are often matched up one-to-one with view models. Simply define it with the same name, and it will automatically be used wherever that view model needs to materialize.

ExtJS users can note that you do not need to worry about defining a class that extends a base component, which also usually requires implementing an `initComponent` method to add sub-objects and containers (like `items` or `toolbars`). GluJS enables you to logically break up your application but still retain the simplicity of declarative JSON.

To make the view "come alive", we need to connect it with the view model. This is done using *binding*.

To understand the purpose of the binding syntax, it is important to take a small step back. ExtJS (among others) provide a rich declarative JSON configuration block for each control. Values provided in the configuration block will only be used on initial render. You must also normally add listeners and invoke setters to add application-specific behavior. In short, there are normally three things you do with any view control:

 * To configure for first-time rendering, provide an initial configuration property in the view itself.

 * To listen for changes from user input, find a reference to the view object and add event handlers.
   * For the button example above, we would normally listen on the 'toggle' event.

 * To change the view object in response to application behavior, find a reference to the view component and call appropriate methods
   * To change the pressed state of the button, we would call the `toggle` method, and to change the state of the title we would call `setTitle`.

Note that there is not necessarily any naming consistency between the three operations. The `pressed` configuration becomes the `toggle` event when you want to listen for changes from the user and the `toggle` method when you wish to change the state from your controller handler. Plus, "finding a reference" is bulky to do correctly.

GluJS radically simplifies this: Just use the configuration properties and throw out the rest. Then through binding, you get the references and other behavior "for free". To bind a control to a view model property or command is simply to provide a string in the following form:

```javascript
controlConfig : '@{viewmodelProperty}'
```

That will tell the GluJS to do three things at once:

 * configure the initial control property with the value from the view model property.
   * So for `pressed:'@{arriving}'` it will set it initially to `true` (the initial value in the view model)

 * add a matching event handler to the control that will change the view model property (or invoke a command) based on user input.
   * In this case, it will add an event handler for the `toggle` event that will automatically set the `arriving` property to match.

 * add a listener to the view model so when its property changes, the view follows suit.
   * If you call `.set('arriving', true)` on the view model, the button will toggle to match.

To sump up, what the binding pattern provides is an extremely simple way to deal with interactivity : view models (controllers) never have to deal with view controls, just with view models. Since view models are just properties, formulas, and commands (functions) that *you define for your application*, what it takes to spell out your application's custom behavior is clear and minimalistic. An entire layer of complex development overhead (finding and maintaining references to view controls, observing them, and manipulating) is done away with.

In addition to the simplicity of binding, GluJS also provides a very straightforward way to "compose" your views dynamically that include "config transformations", localization, one-to-many tab patterns, dynamic areas of the screen, pop-up messages and dialogs, layouts, and more. Furthermore, there is a strong "name convention" piece to GluJS we have not yet introduced that makes your bindings even simpler and more consistent. These will be covered in a later more detailed section.

###Example flow between view model and view

The binding built within GluJS is very simple to use, but lets you quickly compose sophisticated interaction patterns. In the running 'Hello World' example, the flow between view and view model can be summarized as follows:
 1. The button starts out toggled because `arriving` starts out `true`, and the `title` of the panel starts out as 'Hello World!' because `message` (which is calculated from the value of `arriving`) is that value.
 3. When clicked, GluJS sets `arriving` to `false`
 4. That in turn triggers the view model formula called `message` to recalculate based on the provided formula function. It now has the value 'Goodbye World!'
 5. Since the `title` is bound to `message`, GluJS automatically updates the title to match

It's a simple but very powerful pattern. If you are familiar with ExtJS (and even if you are not) it will be useful to walk through the [For ExtJS users: How does this compare?](for-extjs-users-how-does-this-compare.md#for-extjs-users-how-does-this-compare) section below to see how it compares against "straight-ahead" and "MVC" approaches on even a simple reactive application.

###Basic entry points

We have defined the application, but not yet instantiated it anywhere. To do that, GluJS supports two basic entry points.
First is the `Viewport` entry-point when you are using GluJS globally within your ExtJS application:

```javascript
Ext.onReady(function(){glu.viewport('helloworld.main')};);
```

That will locate the `main` view model (defined earlier), find the matching view, and then take over the full page to display the application.

Often within an enterprise you don't have control over the full application and are instead just one module in a bigger framework. To accommodate that within ExtJS, we provide the `glupanel`. Anywhere you can drop a normal ExtJS `panel`, you can drop a `glupanel` and supply what GluJS needs to get started:

```javascript
//within a ExtJS component definition:
items : [{
    xtype : 'glupanel',
    viewmodelConfig : {
        mtype : 'helloworld.main'
    }
}
```

The property `viewmodelConfig` lets you pass in a configuration block in case the parent application needs to hand in initialization parameters into your view model (that's why the alternate object with `mtype` property is used here). You can have as many `glupanel`s as you need in your application and each will be a 'glu root' independent of the others.

It's really that simple. For more information on how that compared with what we are used to doing in ExtJS, continue on to the next section.


*Copyright 2012 Mike Gai. All rights reserved.*