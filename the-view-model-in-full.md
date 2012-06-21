##The view model in full

I promised earlier that we would return to the view model; after all that is the heart of GluJS. Below is a more in-depth walk-through of how a view model comes together.

The view model is the "common sense" representation of application state and behavior. A 'root' view model represents
the application as a whole (or the module if you are a sub-app within a 'portal'), while other view models represent
various screens (tabs, etc.) or areas of a screen.

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

### A note on `this` and scope

Javascript has a different notion of `this` than more traditional Object-Oriented languages. Since functions are first-class and can be passed around and called from another function and are not tied intrinsically to an object instance, the meaning of `this` is determined by the calling context and so can be ambiguous. This often leads to many subtle and not-so-subtle bugs.

One way around this is to set a `me` variable to the appropriate scope and use it instead so that it is enclosed within the function. That means the value of `me` will be fixed. That is certainly an option within GluJS. However, this still means you need to track on a context-by-context basis what the value of `me` is.

GluJS cuts through this ambiguity by *always* making the scope of `this` the containing view model. If you use GluJS as intended, you will never have to provide a `scope` value on any callback and `this` will always have a clear, unambiguous meaning.


### View model parts

The view model is composed of several distinct parts that represent your application state and behavior:

  *    *Properties* - Hold states that various parts of the screen can be in. Usually correspond to things that the user can set
       (like the contents of a text field, or the currently active tab, or which rows of a grid are selected).
  *    *Formulas* - Calculated properties that respond to changes in properties or other formulas. By their nature, they are
       read-only so they typically represent the app 'responding' to user interaction. Glu will analyze the formula and keep it
       updated when input properties change.
  *    *Submodels* - Contains various subscreens and lists of subscreens (glu is for full applications so view models are always
       in a hierarchy with a single root). There is also a special 'parentVM' property to find any view model's container.
  *    *Commands* - Actions that the user can take that aren't represented by simple properties. For instance, a save button or
       hitting the 'close window' indicator.
  *    *Reactors* - Rules that are triggered on property / formula changes so you don't have to put all of your side-effects
       into the property setter. For instance, refreshing the grid when any of several filters change. A formula is really a
       special type of reactor where the action is setting a single property; if it's more complicated, use a reactor.


**Example**

```javascript
glu.model({
   //PROPERTIES
   classroomName : 'Science',
   status : 'OK',

   //FORMULAS
   classroomNameIsValid : function() { return this.classroomName !== '';}
   statusIsEnabled : function(){ return this.classroomNameIsValid;}

   //SUBMODELS
   students : {
       mtype : 'list'
       items : [{
           mtype : 'student',
           firstName : 'Mike'
       }]
   }

   //COMMANDS
   clear : function() {
       this.set('firstName','');
       this.set('status','OK');
   }

   //REACTORS
   when_status_is_not_ok_then_fetch_problem_detail : {
       on : ['statusChanged'],
       action : function() {
           if (this.status!=='OK'){
               //fetch problem detail
           }
       }
   }
});
```

### Properties

Properties are declared simply by adding a property to the config object. The initial value will be whatever is provided.

```javascript
foo : 'we are foo'
```

Properties are accessed through the `get` and `set` methods. You can also read properties by simply reading the backing property directly:

```javascript
this.foo
```

As long as the property is bound or a reactive formulas, the value will always be current with UI state so a getter
is not strictly necessary. It is a matter of preference whether you access the property directly or call

```javascript
get ('foo')
```

to keep the `get` behavior encapsulated within the viewmodel.

To change `get`/`set` behavior (not usually recommended), you can manually add `get`/`set` overrides by using the convention:

```javascript
getFoo : function(){ return...}
setFoo : function(value) {
  this.setRaw('foo',value);
}
```

Calls to `get()` and `set()` will honor these overrides.

In the future, we may provide either automatic getter/setters [`getFoo()` / `setFoo('value')`] and/or Knockout-style getters/setters [`foo()`/`foo('value')`] if there is demand (we have experimented with both)

####Serialization (data)

In addition to declaring properties inline (as shown), you can also provide either a `fields` array or a `modelType` (referencing an object with a fields array like an ExtJS model).

Example of a fields array:

```javascript
defModel ('assets.main',{
    fields : [{name:'id', type:'int'},{name:'description',type:'string'}],
    idIsVisible$: function() { return this.root.options.showIds;}
});
```

The provided fields are added as normal properties to the view model and indeed can overlap with them (you can declare them both in the fields and in the main body normally). The only thing that is different is that they are specially marked by GluJS for serialization.

This lets you do a few special things. First, you can load data into them (deserialize) in one step using a `.loadData` method.

```javascript
    openDetailInspector: function(){
        var detailInspector = this.model({mtype:'asset'});
        detailInspector.loadData(this.assetWithFocus); //copies in current values of the focused asset
        this.open(detailInspector());
    }
```

Secondly, you can serialize them into an object using the `.asObject` method. This will produce an object with only the marked fields, suitable for transfer "over the wire".

In addition, a few more options are surfaced. First, dirty tracking is automatically enabled. Whenever you call `.loadData` (or `.commit` or `revert`), the current values are marked as 'original'. If any of the fields are changed, a global `isDirty` property becomes true. This is a true dirty comparison; if one of the field values is set back manually to the original, the view model is no longer "dirty".

The `commit` function lets you set the current values as the new "original" values, so the data model will no longer be dirty. This is usually done after a save of some sort. The `revert` restores all of the values to the original state (and so the view model is no longer dirty). Note that you can bind a button directly to the `revert` function if you'd like as it is built-in.

With this pattern, your data does not have to be arbitrarily separated into a pure 'model' with no behavior. After all, if you are building a UI you are going to be displaying that data on the screen, and you're going to need all the rich reactive behavior GluJS provides. The view model concept unifies model with controller and makes your architecture a whole lot simpler - and you can still separate out your "data definitions" (models) for re-use by leveraging `modelType` as needed.

### Formulas

Formulas are read-only properties that respond to changes in other properties. To declare a formula, put a `$` at
the end of the name (this won't become part of its name but is just a flag) and then supply a function that returns
a value:

```javascript
saveIsEnabled$: function(){return this.isValid && this.isDirty;}
```

GluJS will scan the function and find property change events to listen for and so will automatically keep up to date with a minimum of recalculation. In the example above, if (and only if) the `isValid` or `isDirty` properties change, it will update the value of `saveIsEnabled`.

Formulas can also be chained: in the example above both `isValid` and `isDirty` are actually other formulas!

#### IsValid

There is one bit of magic-by-convention with formulas. If you name a formula such that it ends with `IsValid`, it will automatically
contribute to a global `isValid` property on the view model. When all such formulas return true, then the global `isValid` will
also be true.

A common usage of this is to disable a 'save' button until the `isValid` (and usually `isDirty`) returns true. For example:

```javascript
save : function(){
   //...
},
saveIsEnabled$ : function(){
    return this.isDirty && this.isValid;
}
```

Here's a trick about isValid functions: you can supply a bit of text instead of `false` and it will count as `false`. This let's you return an error message to a `valid` component binding in a single pass:

```javascript
nameIsValid$ : function(){
    return this.name.substring(0,1)==='Z' ? true : 'The name must start with Z';
}
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

#### Lists and stores

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

###Commands

Whenever the user needs to take an action that isn't necessarily as simple as updating a property - especially when it involves
an Ajax call - then that is a command.

Typical commands are 'save', 'refresh', etc. They are declared simply by providing a function:

```javascript
save : function(){
   //...
}
```

Very often, behavior that could be a command can really be expressed as properties. For instance, the
'collapse' button on a panel could be a 'collapse' command. But it also could be more simply
modeled by a property:

```javascript
detailIsExpanded : true
```

Now you can get both collapse and expand in a single property and a single binding - and you have some state you can use
for other behavior (like in a formula).

Whenever possible, see if what you're trying to do can be reduced to a property and go with that.

####Guard functions

Glu has some strong naming conventions around commands (as well as other properties). These are called *guard functions*. They are simply formulas that *guard* other properties or commands from being accessed and other than the naming convention, are nothing special. The list includes the following (assuming *foo* is the name of a property or command):
 * `fooIsEnabled`  - whether it is enabled/disabled
 * `fooIsVisible`  - whether it is visible/invisible
 * `fooIsExpanded` - whether it is expanded/collapsed

Following the naming pattern not only keeps your code consistent across time and developers, but also enables convention-based binding, a feature that reduces code clutter even further (see
the section on [Binding by convention](the-view-and-binding-in-full.md#binding-by-convention)).

###Reactors

The reactor pattern is simply a shortcut to managing "event observers". It's a powerful way to reduce code clutter and break
out different UI behavior as "rules".  For certain reactive patterns, it lets you "plug in" new behavior in one spot without modifying (and possibly breaking) existing behavior.

Let's say you have an application with a complex grid that needs to be refreshed when any number of text filters, switches and dials on the screen are adjusted. We're going to keep it abstract for now and just call these properties A thorugh E.

A common way to do this would be to separate out the 'refresh' into a function and then have each of the setters for these trigger the refresh:

**Example (NOT RECOMMENDED)**

```javascript
    propertyA: '',
    propertyB: '',
    propertyC: '',
    propertyD: '',
    propertyE: '',
    setPropertyA: function(){
        this.refreshStudents();
    },
    setPropertyB: function(){
        this.refreshStudents();
    },
    setPropertyC: function(){
        this.refreshStudents();
    },
    setPropertyD: function(){
        this.refreshStudents();
    },
    setPropertyE: function(){
        this.refreshStudents();
    },
    setPropertyF: function(){
        this.refreshStudents();
    },
```

In other words, you add the behavior on the triggering end of things. But if there are multiple triggers, this creates redundancies. With GluJS, you could simply state the following instead:

**Example (RECOMMENDED)**

```javascript
    propertyA: '',
    propertyB: '',
    propertyC: '',
    propertyD: '',
    propertyE: '',
    when_a_grid_related_property_changes_then_refresh_grid : {
       on : ['propertyAChanged','propertyBChanged','propertyCChanged','propertyDChanged','propertyEChanged'],
       action : function(){
           this.refreshStudents();
       }
    }
```

Later when you realize that you'd like to load only on an explicit refresh or just need to temporarily suppress the behavior for
debugging, you can just comment it out and "switch off" the behavior in one place.

If you need to add new behavior to some or all of the property events, you can do that as a separate "rule". The ordering is deterministic; GluJS will process in the order you include them in your view model declaration.

While this is an entirely optional pattern, it is a natural and powerful fit for building modular reactive UIs.

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
  *   `localize` - returns the value for the provided localization key and provides the localizer with the current context. See the [Localization](the-view-and-binding-in-full.md#localization) section.
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

###View model mixins
Often different view modesl will share similar structures within an application, or two view models will be closely related but only with a slight twist between them. In those cases, you'll want to use *mixins*.

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


*Copyright 2012 Mike Gai. All rights reserved.*