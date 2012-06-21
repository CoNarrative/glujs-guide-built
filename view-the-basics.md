##View: the basics

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



*Copyright 2012 Mike Gai. All rights reserved.*