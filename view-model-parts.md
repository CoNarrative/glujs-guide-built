##View model: Parts

Below is a more in-depth walk-through of how a view model comes together.

The view model is the "common sense" representation of application state and behavior. A 'root' view model represents
the application as a whole (or the module if you are a sub-app within a 'portal'), while other view models represent
various screens (tabs, etc.) or areas of a screen.

The view model is composed of several distinct parts that represent your application state and behavior:

  *    *Properties* - Hold states that various parts of the screen can be in. Usually correspond to things that the user can set
       (like the contents of a text field, or the currently active tab, or which rows of a grid are selected).

  *    *Formulas* - Calculated properties that respond to changes in properties or other formulas. By their nature, they are
       read-only so they typically represent the app 'responding' to user interaction. Glu will analyze the formula and keep it
       updated when input properties change.

  *    *Sub-models* - Contains various sub-screens and lists of sub-screens (glu is for full applications so view models are always
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

**A note on `this` and scope**

Javascript has a different notion of `this` than more traditional Object-Oriented languages. Since functions are first-class and can be passed around and called from another function and are not tied intrinsically to an object instance, the meaning of `this` is determined by the calling context and so can be ambiguous. This often leads to many subtle and not-so-subtle bugs.

One way around this is to set a `me` variable to the appropriate scope and use it instead so that it is enclosed within the function. That means the value of `me` will be fixed. That is certainly an option within GluJS. However, this still means you need to track on a context-by-context basis what the value of `me` is.

GluJS cuts through this ambiguity by *always* making the scope of `this` the containing view model. If you use GluJS as intended, you will never have to provide a `scope` value on any callback and `this` will always have a clear, unambiguous meaning.

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

to keep the `get` behavior encapsulated within the view model.

To change `get`/`set` behavior (not usually recommended), you can manually add `get`/`set` overrides by using the convention:

```javascript
getFoo : function(){ return...}
setFoo : function(value) {
  this.setRaw('foo',value);
}
```

Calls to `get()` and `set()` will honor these overrides.

In the future, we may provide either automatic getter/setters (`getFoo()` / `setFoo('value')`) and/or Knockout-style getters/setters (`foo()`/`foo('value')`) if there is demand; we have experimented with both and haven't seen a lot of extra value with either though.

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

### Formula Properties

Formula properties (or just formulas for short) are read-only properties that respond to changes in other properties. To declare a formula, put a `$` at
the end of the name (this won't become part of its name but is just a flag) and then supply a function that returns
a value:

```javascript
saveIsEnabled$: function(){return this.isValid && this.isDirty;}
```

GluJS will scan the function and find property change events to listen for and so will automatically keep up to date with a minimum of recalculation. In the example above, if (and only if) the `isValid` or `isDirty` properties change, it will update the value of `saveIsEnabled`.

Formulas can also be chained: in the example above both `isValid` and `isDirty` are actually other formulas!

Formulas are key to managing the reactive nature of your UI application because formulas are themselves 'reactive'. Like formula cells in a spreadsheet, they are always up to date no matter what approach or order the user takes to manipulating the UI. Extremely complex patterns and corner cases can resolve themselves "automatically" when you let formulas manage application flow, so use them whenever you can.

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
the section on [Binding by convention](view-binding.md#binding-by-convention)).

###Reactors

The reactor pattern is simply a shortcut to managing "event observers". It's a powerful way to reduce code clutter and break
out different UI behavior as "rules".  For certain reactive patterns, it lets you "plug in" new behavior in one spot without modifying (and possibly breaking) existing behavior.

Let's say you have an application with a complex grid that needs to be refreshed when any number of text filters, switches and dials on the screen are adjusted. We're going to keep it abstract for now and just call these properties A thorough E.

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


*Copyright 2012 Mike Gai. All rights reserved.*