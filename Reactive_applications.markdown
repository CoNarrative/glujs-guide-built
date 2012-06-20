##Reactive applications

The bar of user experience has been raised. You can no longer deliver a web application with static forms and postbacks that refresh the entire page. Users of enterprise applications expect a rich desktop-like experience that responds to their input and to the back-end and guides them along the proper path. Anything less is frustrating. These are "reactive" applications because they are instantly changing on every input, and any part of the application may trigger a change in any other part. Here are some of the common patterns you might find in a reactive application:
 * Action buttons that enable and disable immediately based on the state of your application and what is selected. For instance, a save button that becomes enabled as soon as a form is "dirty", and then is disabled when not.
 * Areas of a form that enable or expand immediately based on a checkbox being activated or a configuration across multiple fields detected. For instance, a parent configuration checkbox that controls access to dependent options, or a scheduler that changes your choices based on what sort of recurrence you've set.
 * A detail panel that slides in an out based on whether anything is selected
 * Tabs that are automatically selected based on the result of actions. For instance, certain actions may cause a panel to slide out with the relevant tab pre-selected. 
 * An options box that controls validation threshholds for other screens of the app. When changed, any affected screen calculations need to be updated.
 * Calculated fields based on the values of other fields that need to be immediately updated when any of the source fields change.
 * A detail panel that dynamically changes what it displays based on the type and number of rows selected. For instance, it may show a customized truck panel for a truck asset versus a computer asset, or an aggregate chart if multiple items are selected.

If you have any of these sorts of requirements, then you have a reactive application.


*Copyright 2012 Mike Gai. All rights reserved.*