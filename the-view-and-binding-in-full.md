##The view (and binding) in full

The view is the final piece of the puzzline in creating rich, reactive applications.

For the most part, the point of GluJS is that it lets the underlying view provider - for now ExtJS - really shine. GluJS takes care of the high-friction parts - the specification and management of the behavior and enterprise glue - so that your ExtJS can be as straightforward and as uncomplicated as possible. That's good, because even straightforward ExtJS view definitions can take time to master.

The remainder of this section will assume that you are using ExtJS as your view provider within GluJS (since that is all we initially support).

###Defining a view

To define a view, use `glu.defineView` to name a glu after a view model:

```javascript
glu.defineView('helloworld.main',{
    title : 'My first view'
}); //meaning there must be a view model called 'main', unless this is a layout
```

A view defaults to an ExtJS `xtype` of 'panel'. If you want to use a different `xtype`, there is no need to extend a base class. Just use the `xtype` you'd like:

```javascript
glu.defineView('helloworld.personSet',{
    xtype : 'grid',
    title : 'My grid view'
});
```

You can nest ExtJS components within your view as much as you want. The *binding context* of the view will remain the matching view model no matter how deep you go. So you can have a rich view with many nested compenents bound to a flat view model if that suits your application.

Views can also be using a *factory* pattern when using *layouts* (discussed below).

####Includes (quick xtypes)

Sometimes you want to separate a large view up into smaller parts for easier management, even if the view as a whole is simply bound to a single view model.

The typical ExtJS way is to define a new component class and register its alias so that it is available as an `xtype`.

A shortcut within GluJS is simply to define a view as normal and then reference it as a "local xtype" without declaring a global widget. Each application/application module within GluJS has its own distinct namespace, so rather than register globally GluJS takes care of the "local lookup" for you (and keeps widgets from stomping on one another without spelling out a long namespace).

An example of including one view in another:

```javascript
glu.defineView('helloworld.main',{
    title : 'My top level view',
    layout:'hbox',
    items : [{html:'a panel declared inline'}, { xtype : 'aboutCompany'}]
});
glu.defineView('helloworld.aboutCompany',{
    title : 'Imagine a bunch of widgets about us'
});
```


This just "inlines" the declarative JSON into the parent view - simple.

###Materializing a view

As we saw earlier, views are "materialized' automatically by GluJS. You define them, but you don't manipulate them in any way. Instead, they are created and inserted for you.

You begin by using one of two glu components : `glu.viewport` and `glupanel`. The former creates an ExtJS viewport for you, while the latter is an `xtype` usable anywhere within an ExtJS application.

In either case, by using on of them you will be specifying the *root view model* (available as `this.root` from all child view models). Both will materialize the matching root view in the appropriate spot; the first as the viewport of the entire web window, the second wherever you place the panel in a non-glu application.

From there, other views are brought in as appropriate, either as a nested view or as a container-bound view.

####Nested views

Earlier we discussed nested view models and gave the classic example of a master/detail screen relationship. Let's revisit that view model:

```javascript
glu.defModel ('assets.main',{
    assetsList : {
        mtype : 'list'
    },
    detail : {
        mtype : 'asset'
    }
});
glu.defModel ('assets.asset',{
    id : 0,
    name : '',
    //...etc...
});
```

When it comes to the visuals, we know exactly what to do for both - define a view for each of them with the same name, that is one called 'assets.main' and one called 'assets.asset'.
But how do we mark the position of the asset view that corresponds to the 'detail' defined in 'main'? In other words, we have to find out a way to insert the corresponding detail view into 'main'.

In this case, we don't want to include a static view, but want a place to mark: *here's the spot to put the corresponding view for whatever view model I'm currently referencing within the detail property*

We can't simply say "put an asset view here" because what if we had two areas of the screen with a detail and had to disambiguate?

Example of two details:

```javascript
glu.defModel ('assets.main',{
    assetsList : {
        mtype : 'list'
    },
    detail : {
        mtype : 'asset'
    },
    compareDetail : {
        mtype : 'asset'
    }
});
glu.defModel ('assets.asset',{
    id : 0,
    name : '',
    //...etc...
});
```

The solution is simple: bind the ExtJS `xtype` of the sub-item:

```javascript
glu.defView ('assets.main',{
    layout : 'border',
    items : [{
        //GRID or "MASTER"
        xtype : 'grid',
        region : 'center',
        //a bunch of grid definition here...
    },{
        region : 'right',
        //VIEW CORRESPONDING TO DETAIL GOES HERE!
        xtype : '@{detail}'
    }]
});
glu.defView ('assets.asset',{
    xtype : 'form',
    items: [{
        fieldLabel : 'name',
        value : '@{name}'
    },{
        //...etc...
    }]
});
```

Now GluJS knows to nest the corresponding view within the proper location *and* bind it to the nested view model referenced by 'detail'.

Note that you can pass in parent arguments (like 'region') which will override anything set in the defined view. This lets you re-use views across different contexts - the included assets example re-uses an assets view both as inline detail and a pop-up inspector/editor.

GluJS does not yet support "mutable nested views". That is, it currently expects that detail will always remain of `mtype` 'asset'. Since a true mutable view is such a common pattern, that is one of the immediate items on our roadmap. For now, you'll have to put a card view with each of the different possible `mtypes` you might select between and then switch out the card based on the `mtype`. Straightforward actually, but we're always looking to simplify.

There's another common way to materialize a view and that's through container binding - but that is such a powerful feature it is broken out into its own section below.

####Layouts and View Factories

View organization is not always as simple as views and sub-views. In large applications, certain UI patterns emerge. For instance, you may have a standard screen across all your modules that consists of two-side-by side grids and some buttons for common actions. Every module might need to follow this basic layout pattern, but each with its own unique slight twist.

Cutting and pasting the 90% of code that each screen would share in such a scenario is one (ugly) option. Of course you would end up with a very difficult-to-support, inconsistent application.

A logical alternative would be through the ExtJS inheritance model. That would eliminate cut-and-paste code, but introduce difficult-to-follow complexity. You don't want a class tree of widgets to manage; you just want different screens to look the same.

A much simpler approach is one provided by GluJS and modeled after the standard web templating of a *layout*.

A layout is simply an 'abstract' view factory (it is never instantiated directly) that is referenced by actual views. The layout view is what is actually rendered: but it declares "points of interest" that can be supplied by the actual view. The function you provide as the factory accepts the actual view and can now use the properties it provides to populate these points of interest in the layout.

It sounds more complicated than it is because in fact it is dead simple. In the above example, suppose that we have a right grid, a left grid, and a set of action buttons that can be supplied by any particular view. Let's assume that the rest of the screen is supposed to be "locked down" across all conforming application.

It would look something like the following:

```javascript
asset.views.sidebysidelayoutFactory = function(actualView){
    return {
        title : actualView.title + ' Module',
        layout : 'hbox',
        tbar : [{
            text:'~~new~~'
        },{
            text:'~~close~~'
        },{
            text:'~~actions~~',
            menu: actualView.customActions
        }],
        items : [actualView.leftGrid, actualView.rightGrid]
    };
};
glu.defView('asset.assets', {
    parentLayout : 'sidebyside',
    leftGrid : {
        xtype : 'grid',
        store : '@{activeStore}'
    },
    rightGrid : {
        xtype : 'grid',
        store : '@{archiveStore}
    },
    title : 'Assets',
    customActions : [{
        text : '~~archive~~',
        handler : '@{archive}'
    }]
});
```
That is, the 'sidebyside' layout everyone is sharing assumes that your view will provide a `title`, `leftGrid`, `rightGrid` and a set of `customActions` button definitions. As long as those are provided, the layout factory (referenced in `parentLayout` which triggers using the layout) will render the actual view. Now additional views can do the same thing and will all share a common look and feel.

####Dialogs

There is no need to define a view for the view model commands `confirm` and `message`. But when you use the `open` command, that assumes a view model with its own defined view.

Floating windows have a few different configuration options and tend to have fixed heights and widths instead of participating in a layout. Since of the cornerstones of GluJS are views that are reusable in a variety of contexts, we need a way to spell out those special options when the view is in "pop-up" mode.

That is provided by the `asWindow` property in any view:

```javascript
glu.defView('examples.assets.asset', {
    title:'~~title~~',
    collapsed: '@{!expanded}',
    xtype:'tabpanel',
    items:[
        {xtype:'assetSummary'},
        {xtype:'assetSchedule'}
    ],
    tbar : ['save','revert'],
    //settings when in window mode
    asWindow : {
        defaults : {
            header : false,
            border : false
        },
        title : '~~inspectorTitle~~',
        width : 300,
        height : 200
    }
});
```

The `asWindow` block is ignored until the view is triggered through an `open` command in the view model. At that point, the properties within the `asWindow` block are merged into the view itself, replacing any overlapping properties.

###Binding syntax

Of course, views need to be *bound* to a view model in order to have any behavior. This is done through the *binding syntax*.

The binding syntax supports a number of straightforward operators. Keep in mind we want the binding to be concise so that the behavior can be kept in the view model where it belongs. To that end the operators are very simple and concentrate only on *what to bind* (how to locate and process the property) and *how to bind* (also known as "binding directives"). Let's start with the first kind:

 * `!` Inverts a boolean value. Example: `collapsed:'{@!expanded}'`
 * `.` Allows you to naturally traverse into child objects. Example: `text:'{@activeItem.displayText}'`
 * `..`: Find the property at this level or any level above. Example: `save:'{@..save}'` will bind to the save command/function at this view model level and if it cannot find it, walk up the `parentVM` chain until it does find it.

Now for the binding directives (these all come immediately after the `@` sign and before the `{` to indicate that they are about *how* and not *what* to bind.

 * `1` One-time binding - do not listen or update. Example: `value:'@1{displayText}'` will provide an initial value to the control but the control will never affect `displayText` and changes to `displayText` will never affect the `value`.

 * `>` One-way binding - update the view when the control changes, but not vice versa, making the control binding "read-only". Example: `value:'@>{displayText}'` will initially set the value to `displayText` and will track changes to that in the view model, but will never itself update the view model.

 * `?` Optional binding - do not raise an error if the matching view model property is not found. This is usally only used when working with view adapters (extending GluJS) as ordinarily you want to know when you have a "bad binding'. Example: `value:'@?{displayText}'` will let the application continue smoothely even if there is no `displayText` on the view model.

###Binding properties

Binding properties we've already seen many examples of:

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
```

In the example above, we are binding the `pressed` property of a button to the `arriving` property in the view model, and the `title` property of a panel to the `message` property on the view model.

How do we know which properties are bindable? The simple answer is that any configuration property defined for a control within ExtJS is bindable. At a very minimum, it will be initially configured to the initial value of that property - meaning at a minimum it will be a 'one-time' binding.

Many properties support one-way binding (model to control). If the control has a function of the form `setFoo` where `foo` is the name of the configuration property, then there is guaranteed one-way binding support (at a minimum). For instance, the `title` panel config property is bound to the `message` formula property on the view model. Since `setTitle` is a method on the ExtJS `panel`, it is guaranteed to work.

When their is a mismatch in naming between configuration property and the method on the control, that is no cause to worry. GluJS contains binding *adapters* for each component that take care of the mismatch and call the appropriate control method for you (if such a method is available - a few configuration properties really can't be easily changed after initialization).

Controls in turn may update one or more properties they are bound to. Controls derived from *field* for instance always have a way to update their `value` property. Whenever this is the case (and you haven't otherwise indicated in a binding directive), GluJS will automatically also bind from control -> view model. That means whenever the value of the control changes, the matching property on the view model will be updated. This is done through that control's GluJS binding adapter, which listens on an appropriate event emitted by the control and updates the matching view model proprety accordingly.

Control config properties that are available for two-way binding per control are documented in the binding adapter API documentation.

In the example above, we already saw the `title: '@{message}'` binding. Since the title of a panel isn't editable, there is no event to listen on and no logical control -> view model binding. However, the `pressed` state of the button *is* changeable by the user (because it is a toggle button). The `button` adapter for GluJS knows to listen on the `toggle` event from the button; as the user toggles the button GluJS will update the `message` property accordingly.

####Inline text formulas

Here and there it will be convenient to have a facility to 'inline' a text-replacement formula within a view. For instance, you may want a one-way binding that calculates the name of a css class:

An example of hello world that changes the styling of the button:
```javascript
glu.defModel('helloworld.main',{
    arriving : true,
});
//View
glu.defView('helloworld.main',{
    tbar : [{
        text : '~~arrivalStatus~~',
        pressed : '@{arriving}',
        cls : 'button-arriving-@{arriving}'
    }]
});
```

This will make the `cls` change from 'button-arriving-true' to 'button-arriving-false' dynamically (and the GluJS component adapter will set the css class accordingly).

The preferred option is to make the formula explicit in the view model, but this on occasion makes more sense (when there are many of them or when writing a transformer). And of course when the strings are meant to be read, the preferred option is to use the localization facility instead so no text is hard-coded and unlocalizable.

###Binding commands

Sometimes when the user manipulates a control, it isn't so much a state that changes, but a one-time request for some action. For instance, when the user clicks a non-toggle button, they aren't changing a state but just signifying "execute the command attached to the button". Likewise in a grid, when the user double-clicks a row no state actually changes - instead it usually means "open this item up for further inspection".

These are called *commands* within GluJS. As we saw earlier in the view model, a GluJS command is as simple a function on a view model.

ExtJS components raise command-oriented events in two ways. First, through events like `click` and `doubleclick`. Second, through shortcuts like `toggleHandler` and `handler`.

Commands are bound in the same way as properties; by supplying the name of the view model command in the appropriate control config property binding (the event listener or the handler).

Here's an example of both kinds of command bindings:

```javascript
//View model
glu.defModel('assets.main',{
    archiveAsset : function(){
        //invoke an archive operation on the backend
        var ids = this.selectedIds();
        //...etc....
    },
    openAsset : function(model){
        //open an asset detail dialog for viewing/editing
        this.open (this.model({mtype:'asset',id:model.id}));
    }
});
//View
glu.defView('assets.main',{
    xtype : 'grid',
    listeners : {
        itemdblclick : '@{openAsset}'
    },
    tbar : [{
        text : '~~archive~~',
        handler : '@{openAsset}'
    }]
});
```

####Parameterized commands

In the preceding example, `archiveAsset` is a typical  "two-step" action in which you select something (which is tracked in the view model) and then invoke an action (by hitting a button for instance). Since the selection is a separate step up-front, by the time the command is invoked the view model already knows the selections and can use them without being passed in. This is one of the nice ways the view model pattern simplifies the architecture.

Yet sometimes you have a command that only makes sense with a parameter. For instance, the `openAsset` command *selects and invokes* in a single operation; the user can arbitrarily double-click on any row and intend 'open *this one*'. There is no way for the `openAsset` command to rely on a previous state.

In otherwords, the event/command itself is inherently *parameterized*. In those cases, GluJS has a convenient built-in shortcut. ExtJS events that are inherently parameterized always pass in the data target as the second (or later) parameter. GluJS simply passes in every argument but the first into your command (in the case of `itemdblclick` it passes in the `Ext.data.Model` you just double-clicked). That's why the `openAsset` in the example will get the `model` reference it is requesting in its signature.

Another case for parameterization is when you have multiple buttons logically pointing to the same command, each with slightly different options. Rather than creating a unique command function for each, it makes sense to switch out the options as parameters.

To that end, GluJS will automatically append the `value`, `name`, or `itemId` of the control as part of its command invocation.

Example of value-based command parameters:
```javascript
glu.defModel ('assets.main',{
    screenList : {
        mtype : 'list'
    },
    openScreen : function (screenName) {
        this.screenList.add(this.model(screenName));
    }
});
glu.defView ('assets.main',{
    tbar : [{
        text : '~~archives~~',
        handler : '@{openScreen}',
        value : 'archiveSet'
    }, {
        text : '~~live~~',
        handler : '@{openScreen}',
        value : 'openSet'
    }]
});
```

In the provided example, openScreen be called with either 'archiveSet' or 'openSet' as the screenName based on which button is pressed.

GluJS automatically appends the remaining arguments from the calling event as defined in ExtJS *after* it passes in the parameter value (if one exists).

We are planning on adding more explicit parameterization to the binding syntax in the future so that we are not leaning on additional properties in the control that may have other meanings. For instance, something like `@handler:{openScreen('archiveSet')}`. As always, feedback is welcome.

###Container binding (templating)

Static views are usually not all there is in a reactive application. In fact, a very common pattern is managing and rendering a list of items with:

 * a changing list of (possibly mixed in type) models
 * a way to display the list visually by binding to the list and rendering each item through a 'template'
 * (usually) a single 'active' or 'focused' item
 * (sometimes) the ability to select multiple items (for an operation).

ExtJS has some support for this. The most obvious example is the grid, which renders each item (a record or model) in a list (store) - but you have to use grid columns or your own HTML renderers and not ExtJS controls, and it is assumed that all rows are the same. A less-commonly used option is the `DataView`. It is much the same as a grid, but instead of the template being a set of ExtJS components (as one would expect), it is instead a raw HTML `xtemplate` that you supply and again assumes similar rows. Furthermore, since the appeal of a widget set like ExtJS is that you *are not having to write* cross-browser HTML, using the `DataView` can be frustrating and counter-intuitive.

ExtJS also supports normal `container` structures like tabs, accordion layouts, menus, etc. which *do* let you put a variety of items inside them. However, these don't let you bind what they contain to a store. You must manually create and maintain the appropriate child components. This is obviously redundant and repetitive for grids and dataviews; it is equally redundant and repetitive here.

GluJS unifies the 'templated list' pattern into a single, simple concept called `container binding` or more specifically `items binding`.

In short, instead of having limited, manual, idiosyncratic support for rendering a 'templated' list, you can now enable *any* container within ExtJS simply by binding its `items` property to a GluJS list or ExtJS store:

```javascript
glu.defModel('assets.main',{
    assetList : {
        mtype : 'list'
    }
});
glu.defView('assets.main',{
    layout : 'vbox',
    items : '@{assetList}'
});
glu.defView('assets.asset',{
    xtype : 'form',
    items : [{ xtype: 'displayfield'},{xtype:'button}] //etc...
});
```

The preceding example will add an asset form panel (laid out vertically within the main view) for each view model within the assets list. If an asset is added, its corresponding view will show up in the correct spot just like a row in a grid. If the asset is removed from the list, it is removed from the view.

This works for tab panels too:

```javascript
glu.defView('assets.main',{
    xtype : 'tab',
    items : '@{assetList}'
});
```

In short, it works for any ExtJS control that is a `container` and has an `items` property. It also supports common ExtJS shortcuts to single components off of a control. Consider the `tbar` shortcut, which lets you add in an array of button definitions as a short cut to `{xtype : 'toolbar', items : [buttonsArray]}`:

```javascript
glu.defView('assets.main',{
    xtype : 'tab',
    tbar : '@{assetList}'
});
```

####Item Templates

The previous example raises an interesting question: isn't it a bit overkill to force each button to be its own GluJS view? What if I just want a "one-off" template against that record/model/view model?

In the parent container, you can indicate that you want to provide a custom template by providing an object or function as the `itemTemplate`. Let's say I have the items in a menu bound to a list (quite common). Perhaps the concept is that the server provides a list of my favorite shortcut screens (and I need this to fit within a menu instead of looking as a combo box). When you pick a favorite, it adds it as a tab. The definitions would look like this (concentrating on the view side of things):

```javascript
glu.defModel('assets.main',{
    favoriteList : {
        mtype : 'store'
        //more store setup goes here...assume a 'name' field and an 'id' field.
    },
    addScreen : function (screenId){
        //...add the appropriate screen to the list of favorites kept in a tab view
    }
});
glu.defView('assets.main',{
    tbar : [{
        text : '~~screens~~',
        menu : [{
            text : '~~favorites~~',
            menu : '@{favoriteList}' //shortcut definition
            itemTemplate : {
                text : '@{name}',
                value : '@{id}',
                handler : '@{addScreen}'
            }
        }]
    }]
});
```

The 'Favorites' menu item will now follow the contents of the favorites list. When an item is selected, it will invoke a parameterized command (`addScreen`) with the appropriate value parameter. If `favoriteList` is modified, the menu contents will update themselves to match.

Item templates (and item binding in general) are a powerful way to extend ExtJS 'inline'; in many situations it completely replaces the need for custom components.

###Localization

GluJS is an enterprise framework that assumes every application should be localized. Since every application should be localized and all text displayed to the user needs to be localized, it's important to bake that in as an easy-to-use facility.

To make sure text is localized, simply supply a *localization key* instead of the actual text. You may have noticed this pattern throughout the examples - you simply wrap the key in a pair of double-tildes (`~~`). The view above declares three localization keys - `screens` and `favorites`. To supply these, simply provide a localization object for that user:

**English locale object for preceding example code**
```javascript
assets.locale = {
    screens : 'Screens',
    favorites : 'Favorites'
}
```

We recommend that this object be supplied in an appropriately named 'locale' file. However, supplying the appropriate file for the user's locale is outside of the scope of gluJS as a client side library (since sometimes you want to base that on information provided by the server instead of by the client's browser) though it is a fairly simple item to implement.

When you need to localize something within the view model, use the `this.localize` view model method (see the API docs).

The default localizer has a simple but elegant scheme for managing your locale keys. You can keep them all at the root level (`assets.locale`) if you'd like. Or when some seem to correspond more with a particular screen, you can organize them by the names of your view model. For instance, if you have a child view model, you can do the following:

```javascript
glu.defView ('assets.main',{
    title : '~~title~~',
    layout : 'border',
    tbar : [{ text : '~~name~~'}],
    items : [{
        xtype : 'grid',
        region : 'center',
        //a bunch of grid definition here...
    },{
        region : 'right',
        xtype : '@{detail}'
    }]
});
glu.defView ('assets.asset',{
    title : '~~title~~',
    xtype : 'form',
    items: [{
        fieldLabel : '~~name~~',
        value : '@{name}'
    },{
        //...etc...
    }]
});
glu.assets.locale={
    name : 'Name',
    main : {
        title : 'Assets Application'
    },
    asset : {
        title : 'Asset Detail',
        name : 'Asset Name' //overrides the one in the root
    }
};
```

This organization by view model effectively and naturally 'namespaces' your localization keys to avoid conflicts without them becoming long and unwieldy.

####Substitutions (parameterized localization)

Simple text is not always enough - sometimes you need to localize a phrase with arbitrary value substitutions in the middle. You can't simply concatenate in your view model because different languages will order things differently. For that, gluJS supports `string format` like functionality.

When localizing from a view model, along with the key you can pass in values that the key will use in rendering the text. If you want to make sure you can pass in the first name on the message, just do this:

```javascript
//View model
glu.defModel('helloworld.main',{
    arriving : true,
    firstName : 'Mike',
    message$: function() {
        return glu.localize (this.arriving ? 'greeting' : 'farewell', {name:this.firstName});
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
//Locale
glu.helloworld.locale = {
    greeting : 'Hello {name}',  //parameterized with a name
    farewell : 'Goodbye {name}'
};
```

You can use named parameters (shown above) or positional parameters like `{0}`,`{1}`, etc. Through the "magic" of glu formula support, the message will now be recalculated whenever either `arriving` *or* `firstName` changes.

There is currently no support within the view for parameterizing the locale key. However, there is a "backdoor" that lets you access view model properties from within your locale key:

```javascript
glu.assets.locale = {
    removeAssetsMessage:'This will archive {assetSelections.length} asset(s). Would you like to continue?'
}
```

Will work even if you don't provide the key, assuming `assetSelections` is an array property on the view model. Keep in mind that this is probably not the best way to organize things because it forces your locale keys to be somewhat "viewmodel-aware" but is provided as an option for corner-cases.

####Custom localizer

If you already have a localization scheme in place at a higher level of your application, or prefer a different organization, you can replace the default localizer with one of your own by providing a function to `glu.setLocalizer(function(config){})`. The provided function will be supplied a `config.key`, a `config.ns`, a `config.viewmodel` and a `config.params` for each localization call and you can then locate your localized text as needed.

###Binding by convention

We've up until now covered "explicit" binding of control config properties. This already is a powerful pattern for organizing UI and greatly simplifying code. Yet as you use it, you will inevitably notice that binding patterns tend to repeat themselves.

Consider the case of a set of fields in a form with some standard commands in the toolbar:

```javascript
glu.defView ('assets.asset', {
    xtype : 'form',
    tbar : [{
        text : '~~save~~',
        handler : '@{save}',
        disabled : '@{!saveIsEnabled}
    },{
        text : '~~revert~~',
        handler : '@{revert}',
        disabled : '@{!revertIsEnabled}
    }],
    items : [{
        fieldLabel : '~~name~~',
        value : '@{name}',
        disabled : '@{!nameIsEnabled}',
        valid : '@{nameIsValid}'
    }, {
        fieldLabel : '~~status~~',
        value : '@{status}',
        hidden : '@{!statusIsVisible}',
        valid : '@{statusIsValid}'
    }, {
        xtype : 'fieldset',
        collapsed : '@{!archiveSectionIsExpanded},
        items : [{
            fieldLabel : '~~lastInventory~~',
            value : '@{lastInventory}'
        },{
            fieldLabel : '~~recoveredLicenses~~',
            value : '@{recoveredLicenses}',
            hidden : '@{recoveredLicensesIsVisible}'
        }]
    }//...etc....
});
```

These are very common bindings for rich forms with validation and fields dependent on the values of other fields. Since we're following the naming convention established earlier for [guard functions](guard-functions), the names are deterministic based on the property we are binding to.

Here's the magic shortcut: provide a `name` property with the name of the appropriate property or command (the name only, not a binding) and let gluJS apply the bindings for you:

```javascript
glu.defView ('assets.asset', {
    xtype : 'form',
    tbar :[{
        name : 'save'
    },{
        name : 'cancel'
    }],
    items : [{
        name : 'name',
    }, {
        name : 'status',
    }, {
        xtype : 'fieldset',
        name : 'archiveSection',
        items : [{
            name:'lastInventory'
        },{
            name: 'recoveredLicenses'
        }]
   }//...etc....
});
```

Each ExtJS control adapter has a set of 'conventions' that it will apply based on the nature of the control. Some are standard across all components (like `hidden`). Others are specific to certain components (like `fieldLabel` and `collapsed`). These are all detailed in the API documentation. When GluJS sees a 'name' property, it applies the bindings automatically using the optional flag - so if the matching view model property isn't there it just ignores them. You can always override the convention by explicitly setting the component property to whatever you want.

Convention-based binding is not just a shortcut - it's a way to even further simplify your development workflow to "stay in the zone" of setting up story-based specification tests and then implementing the view model.

Imagine that you want to add a validator to the `lastInventory` property. Add the appropriate steps in your story, and let the test fail so that you know the test is valid. Then supply the formula property according to naming convention:

```javascript
    //viewmodel...
    lastInventoryIsValid$:function(){
        return this.date > this.minDate ? true : 'Too far back';
    }
```

Your test now passes. Fire up the application and the view behavior will be already "wired" in because you followed the convention and used convention-based binding. Repeat for the next bit of functionality.

We haven't gotten to GluJS extension points yet, but it's worth mentioning that this pattern enables even more than a simplified workflow. Convention-based bindings are just one example of "cross-cutting" interceptors within GluJS. Since it is straightforward to add your own (see [Extending GluJS](extending-glujs.md#extending-glujs)), it becomes a powerful means to enforce application consistency. You can use that single key not only for localization, but to determine even the type of control to use (within a field) or whether to even include the control based on security permissions. These become 'application' level concerns so that they are enforced automatically instead of being enforced ad-hoc by each developer when they remember - the rule in GluJS application design is that if some feature appears everywhere, then it probably should appear nowhere and be removed to infrastructure.

Name-based bindings come with one final shortcut: if you just provide a string in an `items` array, it assumes you mean a configuration of the form `{ name: 'stringYouProvided'}`. So the previous view can be simplified further.

This shortcut is especially handy when you would like to automate button and control generation:

```javascript
glu.defView ('assets.asset', {
    xtype : 'form',
    tbar :['save','cancel'],
    items : ['name','status',{
        xtype : 'fieldset',
        name : 'archiveSection',
        items : ['lastInventory','recoveredLicenses']
   }//...etc....
});
```

Plus it's just a whole lot simpler to read.

We recommend that you lean heavily on your application as it not only makes your code much more concise, it gives you the ability to logically "build your UI" at run-time based on cross-cutting enterprise concerns - like 100% consistent controls, localization, and security.


*Copyright 2012 Mike Gai. All rights reserved.*