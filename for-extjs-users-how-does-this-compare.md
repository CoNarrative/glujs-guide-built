##For ExtJS users: How does this compare?

The View Model pattern is really just a logical evolution of existing patterns you may have already seen. We'll use ExtJS code as a basis but this applies to any component library.

You may be familiar with the "standard" "inline component" pattern in simple ExtJS examples. Let's start with the "hello world" example using that approach:

**Inline component approach (NOT GluJS!)**

```javascript
//old-school Ext JS straight-ahead style
Ext.onReady(function (){
    //Starts with view definition
    new Ext.Viewport({
        layout : 'fit',
        items : [{
            title : 'Hello World!',
            tbar: [{
                text : 'Coming/Going',
                //'controller' code that responds to the user toggling the button
                toggleHandler : function(button, state){
                    var msg = this.state ? 'Goodbye World!' : 'Hello World!';
                    button.ownerCt.setTitle(msg); //have to find some way to reference the other control
                }
            }]
        }]
    });
});
```

The problem with this approach is that there is no separation of concerns. The application is one intermixed "blob" of view and behavior. It looks fine for a very small application. But following this style on a full enterprise application leads to a dense tangle of nested spaghetti code. Every component pokes values into every other component (just as the toggleHandler is pushing into the parent panel's title) and the system behavior becomes incredibly hard to track and maintain over time.

Just as importantly, there's no clean way to test the custom behavior you've added without a great deal extra infrastructure work (like installing and writing Selenium tests in a lab, or attempting to run everything in a Node server with a non-rendered DOM, etc.)

The next logical improvement is to separate the view (the actual control definition) from the controller (the logical behavior). That at least will lay down some logical file organization and make behavioral code easier to centralize and manage. This approach is exemplified by the MVC pattern offered in Ext JS 4.x.

**MVC approach (NOT GluJS!)**

```javascript
//CONTROLLER FILE
Ext.define('Helloworld.controller.Main', {
    extend: 'Ext.app.Controller',
    refs: [
        {
            ref: 'mainPanel',
            selector: 'main'
        }
    ],
    init: function() {
        //Component Query to "wire up" toggle handler
        this.control({
            'main#exitToggle': {
                toggle : this.onButtonChange
            }
        });
    },
    onButtonChange: function(button, state) {
        var msg = this.state ? 'Goodbye World!' : 'Hello World!';
        this.getMainPanel().setTitle(msg); //now getting a reference through the component query defined in refs
    }
});
//VIEW FILE
Ext.define('Helloworld.view.Main', {
    extend: 'Ext.panel.Panel',
    alias: 'main',
    initComponent: function() {
        this.tbar = {
            xtype: 'tbar',
            items : [{
                itemId : 'exitToggle',
                text : 'Coming/Going' //we let the controller wire up the toggle handler
            }]
        };
        this.callParent();
    }
});
//VIEWPORT FILE
Ext.define('Helloworld.view.Viewport', {
    extend: 'Ext.container.Viewport',
    layout: 'fit',
    requires: [
        'Helloworld.view.Main'
    ],
    initComponent: function() {
        this.items = [{
            xtype: 'main'
        }];
    }
});
//APPLICATION BOOTSTRAP FILE
Ext.application({
    name: 'Helloworld',
    autoCreateViewport: true,
    controllers: ['Main']
});
```

This is definitely some organizational improvement. Following a MVC pattern at least puts the code into separate code files, and now the application will have some basic 'lines' instead of being a complete 'blob'. But it does this *at the expense of much more code* (even just considering the view and controller).

In effect, all we've done is externalized how component event handlers and references back to other components are "wired". Now that we have split view and behavior apart, we have *even more work* than before in bringing them together again. A more accurate name for the pattern would be MVCRCQ - a model, view, controller, references, and component queries, because without `refs` and component queries for each and every bit of behavior, the MVC pattern doesn't work.

Because the view and controller still need to be manually "wired up", there has been little gain at an architectural level:

 * The app is still "brittle": the references and component queries mean your controllers have to be deeply aware of your view tree.

 * There is still no incremental way to concentrate just the critical behavior without simultaneously having to get every last view component also correct.

 * There is even more code to write

 * And there is still no obvious "entry point" to begin testing.

###The GluJS way

The best of both worlds would be if behavior could both be cleanly separated into a controller, yet somehow reconnect to the matching view automatically without costly wiring. All the interesting state could be put into the controller and within the controller we would never have to deal with the view at all. This approach would offer these benefits:

 * You can eliminate all of the "hand-wiring" and dramatically shrink/simplify/stabilize code.

 * The controller and view are entirely stand-alone and separate, greatly reducing the "brittleness" of too many interconnecting parts.

 * You can start defining, building, and testing custom application behavior immediately.

A controller that does this - models important application state and doesn't reference the view - is simply called a "view model."

This of course is the exact approach we take in GluJS:

```javascript
//View model
glu.defModel('helloworld.main',{
    arriving : true,
    message$: function() {
        return this.arriving ? "Hello World!" : "Goodbye World!"
    }
});
//View
glu.defView('helloworld.main',{
    title: '@{message}',
    tbar : [{
        text : '~~arrivalStatus~~',
        pressed : '@{arriving}'
    }]
});
//ExtJS application bootstrap
Ext.onReady(function(){glu.viewport('helloworld.main');});
```

The behavior is cleanly separated, we do it without introducing any bloat, and best of all, it is entirely testable.


*Copyright 2012 Mike Gai. All rights reserved.*