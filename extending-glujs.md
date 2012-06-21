##Extending GluJS

GluJS is itself built as a series of simple stacked patterns; the 'convention-based binding' covered in a previous [section](view-binding.md#binding-by-convention) is a great example since all it does is walk the view configs, look for a 'name' property, and then modify the configuration by adding some standard bindings. You could build the same thing yourself within GluJS fairly easily.

Those bindings in turn reduce to a standard pattern of 'find the event I can attach to for changes from the component' and 'let me invoke the property method on the component'.

All of these operations can be reduced to a simple concept of `view adapters`. View adapters are special javascript singleton objects that tell gluJS how to either

  1. normalize binding on a per-control, per-property basis (*component adapters*) or
  2. modify the configuration block to enforce some sort of pattern for this component (*transformer adapters*) or
  3. do the same across all components (*plugin adapters*)

In fact, a good chunk of GluJS intelligence resides in the adapters - the GluJS core is pretty straightforward. Even though there are three types of adapters, they all share the identical API. The different adapter types refer to *roles* the adapters play and when they are invoked, not to any essential difference.

###View component adapters

Let's take a look at a fairly simple component adapter:

**Example: gluJS button adapter (stripped of comments)
```javascript
glu.regAdapter('button', {
    extend : 'component',
    defaultTypes : {
        menu : 'menu'
    },
    isChildObject : function(propName){
        return propName==='menu';
    },
    menuShortcut : function(value) {
        return {
            xtype:'menu',
            defaultType:'menuitem',
            items:value
        };
    },
    applyConventions: function(config, viewmodel) {
        Ext.applyIf (config, {
            handler : glu.conventions.expression(config.name,{up:true}),
            pressed : glu.conventions.expression(config.name + 'IsPressed', {optional:true}),
            text : glu.conventions.asLocaleKey(config.name)
        });
        glu.provider.adapters.Component.prototype.applyConventions.apply(this, arguments);
    },
    pressedBindings:{
        setComponentProperty:function (newValue, oldValue, options, control) {
            control.toggle(newValue, true);
        },
        eventName:'toggle',
        eventConverter:function (control) {
            return control.pressed;
        }
    }
});
```

We're not going to go into all of the details - that's covered in the API docs. More important is to get a feel for how GluJS is structured and how you can extend as needed.

This is a component adapter for the `Ext.button.Button` component. Here are some of the parts you're seeing here:

 * `glu.regAdapter('button',` - we name component
 * `extend : 'component'` - inherits from the `component` adapter. You can chain adapters to match the control hierarchy of the widget set. Inheriting from `component` means we'll get the standard naming conventions and bindings.
 * `defaultTypes` through `menuShortcut` - tells gluJS about ExtJS definition shortcuts so that it can walk the view tree properly
 * `applyConventions` - applies conventions when gluJS detects a `name` property. Notice that the `handler` is a required binding (if you call a button 'save' there better be a 'save' command on the view model) while 'pressed' is an optional binding.
 * `pressedBindings` - provides special helpers when binding to the `pressed` component property, such as how to set the component property in response to view model changes (`setComponentProperty`) which event to respond to `toggle` and how to return the event value `eventConverter`.

Whenever we come across a component property (or component) that we have yet 'gluified', we simply go into the appropriate adapter (or create a new one) and add the bindings. In other words, instead of having 'hacks' spread throughout the code to remember that you have to call 'toggle' to change the 'pressed' state of a button, we set it up one time in the adapter and then can ignore an unnecessary 2/3 of the ExtJS control API (along with the other GluJS benefits).

If you have a custom control of your own, you can just make an adapter as above and register it. Even simpler, make sure you controls have a consistent `foo:value`/`setFoo()` pattern and you will minimize your need for adapters in the first place.

###View transformer adapters

Ever need to create a custom control that was just a collection of controls or some simple tweaks, and ended up having to make a whole complicated ExtJS component class? Ever wanted a plugin that could change everything about the control, even including its xtype? That would be obviously impossible within ExtJS, since plugins are invoked *after* a specific component is created.

What you would need is the ability to transform the configuration block itself *before* ExtJS started making components off of it. That's all a transformer is. It lets you stick with the most straightforward part of ExtJS - the configuration properties we use when setting up views - and avoid the internal complexities and limitations of sub-classing a component.

Here's an example of a plugin provided with gluJS that actually determines the xtype of a property by a set of rules *based on the data type of the property*. In other words, if the property is a date / time, you get a date time picker, a boolean becomes a checkbox, a field constrained by another becomes a combo-box, etc.

**Example of an 'autofield' transformer adapter (code collapsed for clarity)**
```javascript
glu.regAdapter('autofield', {
    beforeCollect: function (config, viewmodel){
        var key = config.name;
        //lookup key in the data definition
        //...
        //build config based on data type
        //...
        else if (field.type == 'integer' || field.type == 'int' || field.type == 'number') {
            config.xtype = 'numberfield';
        }
        else if (field.type == 'boolean') {
            config.xtype = 'checkbox';
        //...
    }
}
```

To use a transformer that will decide the xtype, we simply use it *as* an xtype. So now when we do this in that earlier example:

```javascript
glu.defView ('assets.asset', {
    xtype : 'form',
    tbar :['save','cancel'],
    defaultType: 'autofield',
    items : ['name','status',{
        xtype : 'fieldset',
        name : 'archiveSection',
        defaultType: 'autofield',
        items : ['lastInventory','recoveredLicenses']
   }//...etc....
});
```

Each field will *automatically* get its proper type based on its data value. When designing enterprise applications, this gives you the ability to *standardize the feel of the application* by enforcing control consistency. Want all of your booleans to be a yes/no radio button instead of a checkbox? No problem, just change the `autofield` adapter and you can enforce the change globally throughout the application. Of course, wherever exceptions are needed, you can fall back to explicit `xtype`s (as shown in the `fieldset` definition above).

Perhaps you know the xtype, but you would like to transform it based on a pattern. For instance, suppose you had a notion of "change tracking" in a field in which you needed to see the original value of the control laid out in a display field next to the current value. Since this is a fundamental change in each field, you might be tempted to make a new type of control per field type ('changeawareradiobutton', 'changeawaretextfield', etc.), or make a special type of container that throws out the controls you add to it and rebuilds the structure. But not only would that be a great deal of custom work, you couldn't quickly change back to contexts in which you *didn't* need that capability. In short, you don't have many good options.

With the ability to intercept the config *before* rendering and the ability to specify a `transform` adapter at any point, you can handle these sorts of enterprise scenarios quite simply. Just make an adapter that modifies the config tree (perhaps to a `fieldcontainer` or a `compositefield`) just like the autofield adapter. Since you're just editing the config in code there is no ExtJS or any other magic you need to know.

Then to apply it use the `transforms` property (there's a `defaultTransforms` shortcut similar to `defaultType`):
```javascript
glu.defView ('assets.asset', {
    xtype : 'form',
    defaultTransforms : ['changeaware'],
    items : ['name','status']
   }
});
```

Now the fields will have the transform applied. You can chain transform adapters - when you use one in the xtype it goes first, followed in order by any provided in the transforms list. Therefore the following will both determine the appropriate `xtype` for each component, it will also then transform that component into a 'change aware' version.

```javascript
glu.defView ('assets.asset', {
    xtype : 'form',
    tbar :['save','cancel'],
    defaultType: 'autofield',
    defaultTransforms:['changeaware'],
    items : ['name','status',{
        xtype : 'fieldset',
        name : 'archiveSection',
        defaultType: 'autofield',
        defaultTransforms:['changeaware'],
        items : ['lastInventory','recoveredLicenses']
   }//...etc....
});
```

To remove the 'change aware' behavior you would simply remove the `defaultTransforms` property, making it easy to add or remove this behavior at any point in the application without disturbing your work.

###View global adapters

Sometimes you want that adapter to run *everywhere* without the developer having to deal with. A common example of this would be a permissions-applier adapter that would edit the view tree based on what the user is permitted to do. You don't want the developer to have to worry about; instead you want to inspect all of the view components as they materialize. If you are using `name` consistently, then you have everything you need to enforce a permissions structure based on view model and property or command.

 ```javascript
glu.regAdapter('permissions', {
    beforeCollect: function (config, viewmodel){
     var areaKey = viewmodel.viewmodelname;
     var specificKey = config.name;
     if (!assets.permitted(areaKey,specificKey) {
        config.disabled = true;
     }
     //in this adapter we disable, but we could remove, hide, etc...
    }
}

 glu.plugin ('permissions');
 ```

The `glu.plugin` function lets you take any adapter and run it on every control in the pipeline.

In short: GluJS adapters let you enforce core enterprise concerns in a concise, simple, repeatable manner.


*Copyright 2012 Mike Gai. All rights reserved.*