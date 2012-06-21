##View model: Putting the pieces together

In the previous section we talked about the various parts that go into a view model. Here we go over how to define, create, and compose them to form an application.

###Defining and creating a view model

A view model can be defined and instantiated on the fly from within a view model:

```javascript
var model = this.model ({
   status: 'OK'
});
//or for a dialog
this.open ({
   status: 'OK'
});
```

or it can be defined first (with namespace) and then referenced later through the 'mtype' property (as long
as you are in the same namespace):

```javascript
glu.defModel('example.main', {
   status : 'OK'
});
var vm = this.model ({
   mtype : 'main'
});
```

A view model can also be defined 'inline' within a containing view model using an 'mtype' property of 'viewmodel':

```javascript
glu.defModel('example.main', {
   detail : {
       mtype : 'viewmodel',
       myProp : 'A'
   }
});
```

or simply by reference

```javascript
glu.defModel('example.subscreen', {
   myProp : 'A'
});
glu.defModel('example.main', {
   detail : {
       mtype : 'subscreen'
   }
});
```

Referenced view models are fully parameterizable, so you can initialize any of the values with overrides:

```javascript
glu.defModel('example.main', {
   status : 'OK'
});
//...later...
var vm = this.model({
   mtype : 'main',
   status : 'BAD'
);
```

A 'root' view model can be instantiated by one of several entry points, but most typically by leveraging the glu viewport factory within the ExtJS entry-point:

```javascript
Ext.onReady(function(){glu.viewport('example.main');});
```

or

```javascript
Ext.onReady(function(){
    glu.viewport({
       mtype :'example.main',
       //optional parameters...
    });
});
```

or it can be included as a subpanel of an already existing application panel:

```javascript
//...
items : [{
   xtype: 'glupanel',
   viewmodelConfig : {
       mtype : 'main',
       //optional parameters...
   }
}],
//...
```

or (usually just for testing) you can start one with a fully qualified namespace

```javascript
var vm = glu.model('example.main');
```

### Submodels / child view models

GluJS is a framework for quickly developing real applications with complex navigation and screens. Very often you'll want to split your
view models in parts. The initial example above has a list of 'student' view models. This list could correspond on the screen to
a set of items in a mobile list or a set of tabs. This is just one of the built-in UI composition patterns within GluJS.

Submodels are indicated by using the `mtype` property within a nested object.

Note that you *do not* have to divide your view model up just because your screen has logical areas. It can often remain relatively flat if the interactions are simple enough. For instance, you may have a form with multiple tabs representing different parts of a large data object. You can keep it as one large flat data object, and just break it up at the view level.

Other times, you will have an obvious sub-model. The most common of these is a master-detail page in which there is a grid or list and another area with a form that lets you view and/or edit the detail.

That might look like the following:

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

### Lists and stores

You might have noticed above that `mtype` for assetsList is 'list'. The GluJS list object fills in a gap left by frameworks like ExtJS. Many times you do not need a full `store` with support for proxies, flattening of json objects through a reader, and the other facilities provided by a store. You just need a simple observable list that can notify components when an item is added or removed. That's exactly what the 'list' provides.

While a list can contain any value, the most powerful use for a `list` class is when the list contains view models of its own. That naturally can be bound to a set of tabs, cards, or even panels laid-out horizontally or vertically on the screen ('repeaters'). For instance, if you want to dynamically add tabs (we'll call them *screens* below) based on different selections of the user, then the view model might look like the following:

```javascript
glu.defModel ('assets.main',{
    screenList : {
        mtype : 'list'
    },
    openScreen : function (screenName) {
        this.screenList.add(this.model(screenName));
    },
    openScreenIsEnabled$ : function(){
        return this.screenList.length < 8;
    }
});
```

If a tabpanel's `items` collection is bound to the screenList, it will automatically construct the matching tab for that type of view model. Conversely, when the view model is removed from the list, so is the tab from the tabpanel. As a bonus illustration, the example also includes a cap on how many screens can be active; if there are already 8 or more screens open, openScreen will be disabled until enough are closed.

### The view model graph

When you create child view models, those are automatically parented as follows:
 * `this.parentVM` - the child is given a reference to the parent view model. If the child is part of a `list`, the parentVM is the parent of the list, not the list itself. Note that this can change if the view model is 'reparented'.
 * `this.parentList` - the parent list of a child in the list. This too can change based on what list the child is added to.
 * `this.root` - the root view model. The root of the entire view model tree. This is immutable and is simply the "initial entry point". In other words, it constitutes the "application" as far as glu is concerned. That does not preclude you from having multiple application roots within a single page (such as different modules in a larger non-glu application).

Reactors and formulas are automatically attached/detached based on the current configuration of the tree. For instance, if you create a view model with a formula that references `this.parentVM`, when detached (e.g. removed from a list), that view model will stop receiving notifications until reattached at the same point or elsewhere.

The idea of a tree of collaborating view models is a key organizing pattern to larger applications. It lets you break down the application into component parts, while letting them naturally reference one another without having to worry about explicit observer patterns. You can even think of it as giving you a natural fine-grained message bus - put properties or events you are interested in globally at the root level, and more local properties/events lower down the chain.

###View model mixins
Often different view models will share similar structures within an application, or two view models will be closely related but only with a slight twist between them. In those cases, you'll want to use *mixins*.

Whenever you define a view model, you can provide an array of other view models under a `mixins` property as follows:

```javascript
//the base mixin
glu.defModel('assets.asset', {
  name : '',
  //..other shared commands, functions, and properties...
});
glu.defModel('assets.activeAsset',{
  mixins : ['asset'],
  //...unique properties, commands, functions
});
glu.defModel('assets.archivedAsset',{
  mixins : ['asset'],
  //...unique properties, commands, functions
});
```

One of the design goals of GluJS was to avoid the complexity of inheritance patterns within Javascript and instead favor object composition. There is no support for chaining method calls to a parent 'class'. Whenever you run into situations where you might be tempted to use inheritance, there's usually another way to break down the problem. A common technique is a 'virtual method' such as having the mixin call another, absent function provided by the target view models:

```javascript
//the base mixin
glu.defModel('assets.asset', {
    save : function(){
        this.saveInternal();   //'virtual' method
    }
});
glu.defModel('assets.activeAsset',{
  mixins : ['asset'],
  saveInternal : function(){
  }
});
glu.defModel('assets.archivedAsset',{
  mixins : ['asset'],
  saveInternal : function(){
  }
});
```

This approach avoids class definition trees and keeps things simple.

### Convenience methods

There are a number of convenience methods that are commonly used within a view model. Use them
instead of the matching ones on the 'glu' object because they
  *    pass in the local namespace
  *    set the scope of any callback to the viewmodel (so 'this' always refers to the view model)
  *    automatically create parent/child associations where appropriate
  *    are automatically mocked as needed by the simulation/testing framework just on that view model without breaking core 'glu' for other view models.

The methods are as follows (please refer to the API docs for details):

  *   `ajax` - makes a call that is scoped to the view model and has a few convenience hooks
  *   `model` - makes a child model that is appropriately parented to the current view model
  *   `localize` - returns the value for the provided localization key and provides the localizer with the current context. See the [Localization](view-the-basics.md#localization) section.
  *   `confirm` - asks for a confirmation from the user
  *   `message` - returns a message to the user
  *   `open` - opens a user pop-up screen that is itself a view model

### Dialogs

There are three ways to open up a dialog to the user.

First, if it is simply a message box, then use the built in `this.message` on the view model. It is (for now) simply an alias on to the ExtJS MessageBox functionality that scopes the callback to the viewmodel.

```javascript
    //...in view model definition
    launch : function() {
        this.message (this.localize('launching', this.instanceSelections))
    }
```

Second, if it is a confirmation, use the `this.confirm`, which has the same syntax as within ExtJS.
```javascript
    removeAssets:function () {
        this.confirm({
            title:this.localize('removeAssetsTitle'),
            msg:this.localize('removeAssetsMessage'),
            buttons:Ext.Msg.YESNOCANCEL,
            icon:Ext.Msg.QUESTION,
            fn:function (btn) {
                if (btn !== 'yes') return;
                this.removeAssetsActual();
            }
        });
    },
    removeAssetsActual:function () {
        this.ajax({
            url:'/json/assets/action/remove',
            params:{ids:this.selectedIds()},
            success: this.refreshAssetList
        });
    },
```

The naming pattern we use for a confirmation of a command is to call the first step the actual command ('removeAssets') and then the second step after being confirmed simply 'removeAssetsActual'.

We are in fact looking into formalizing that as an automatic GluJS pattern - feed back is welcome.

Finally, opening up a full pop-up window in which you have complete control - such as a wizard, a modal edit window, or a configuration screen - is done through then `open` function:

```javascript
glu.defModel('assets.main', {
    options : {mtype:'options'},
    openOptions : function() {
       this.open(this.options);
    }
});
glu.defModel('assets.options', {
    warnings : true,
    offMaintenanceWarning : false,
    missingWarning : false,
    offMaintenanceWarningIsEnabled$ : function(){
        return this.warnings;
    },
    missingWarningIsEnabled$ : function(){
        return this.warnings;
    }
});
```

When using `open`, you can either provide a view model instance request ( such as `{mtype:'options'}` ) in which case the view model will last until closed, *or*, you can provide an already existing instance so that it persists. In the example above, we're doing the latter so that the 'options' view model persists.

To close an opened view model, simply call its `close` method. There is also a private `doClose` method available to the view model which does the actual work. That lets you easily override the close method without inheritance. However, for the common pattern of vetoing a close under certain circumstances, we recommend simply proactively guarding the availability of the close command. Assuming that you want to veto closing of an options screen until all values are set, the best way is the simplest:

```javascript
glu.defModel('assets.options', {
    closeIsEnabled : function(){
        return this.isValid; //only allow closing once all options are initially configured
    }
});
```

Binding adapters within GluJS will control the accessibility of any 'close' buttons to match the state of `closeIsEnabled`.


*Copyright 2012 Mike Gai. All rights reserved.*