##GluJS

GluJS is a lightweight specification-driven reactive UI framework for building rich enterprise web and mobile applications using HTML5 widget libraries like Sencha ExtJS.

###Reactive applications

The bar of user experience has been raised. You can no longer deliver a web application with static forms and postbacks that refresh the entire page. Users of enterprise applications expect a rich desktop-like experience that responds to their input and to the back-end and guides them along the proper path. Anything less is frustrating. These are "reactive" applications because they are instantly changing on every input, and any part of the application may trigger a change in any other part. Here are some of the common patterns you might find in a reactive application:

 * Action buttons that enable and disable immediately based on the state of your application and what is selected. For instance, a save button that becomes enabled as soon as a form is "dirty", and then is disabled when not.

 * Areas of a form that enable or expand immediately based on a checkbox being activated or a configuration across multiple fields detected. For instance, a parent configuration checkbox that controls access to dependent options, or a scheduler that changes your choices based on what sort of recurrence you've set.

 * A detail panel that slides in an out based on whether anything is selected

 * Tabs that are automatically selected based on the result of actions. For instance, certain actions may cause a panel to slide out with the relevant tab pre-selected.

 * An options box that controls validation thresholds for other screens of the app. When changed, any affected screen calculations need to be updated.

 * Calculated fields based on the values of other fields that need to be immediately updated when any of the source fields change.

 * A detail panel that dynamically changes what it displays based on the type and number of rows selected. For instance, it may show a customized truck panel for a truck asset versus a computer asset, or an aggregate chart if multiple items are selected.

If you have any of these sorts of requirements, then you have a reactive application.

###Why GluJS? The short version

Developing rich, reactive business-oriented web UIs turns out to be much more difficult than it first appears. For most enterprises, the smartest thing to do is to find a library that offers a rich UI component set - like Sencha ExtJS - to handle the complexities of laying out UI across browsers.

That's an excellent approach, but it doesn't necessarily get the job done. Tools like Sencha ExtJS offer steep learning curves even for experienced developers. Organizing large applications with rich reactive behavior can quickly lead to a tangle of code unless you are already an expert. Even with teams experienced in the tool, there isn't always an obvious way to organize the code so that it is concise and manageable and side-effect free.

Most problematic of all, there is no obvious out-of-the-box pain-free way to make the code testable as you go. The richer the application, the more you end up plugging holes.

GluJS is for anyone who is trying to build a rich web HTML UI using a declarative widget toolset like Sencha ExtJS. It dramatically simplifies the architecture and reduces code to a minimum while getting out of the way to let ExtJS shine. Non-experts can now be productive, while experts can drop all of the boiler-plate code and concentrate on the application itself.

Best of all, it is "test-driven" from the ground up so that all of your UI code -- from user interactions to AJAX calls -- can be simulated in a fast and developer-friendly test harness.

If you want more detail on the challenges GluJS addresses, continue with the "Extended Version"; otherwise skip on to the "SVVM" section.

###Why GluJS? The extended version

Developing rich web and mobile HTML-based applications in Javascript can be new territory for many enterprise development teams. While there are great libraries like Sencha ExtJS to make a "desktop-like" experience, even ExtJS experts face significant challenges turning the initial "it looks great!" ExtJS wireframes into an actual large working application:

####Difficult to test
It's not easy to figure out where and how to build unit tests that cover rich UI interactions. That often means tests are done last using Selenium-style tools if they are done at all. Lack of automated test coverage means extra cycles figuring out what the stakeholders really want, and permit additional new features and bug fixes to break previously working code.

####Adding rich reactive behavior gets complicated fast
Laying out a set of components is one thing. But modern applications need to do more than look good. They have to react in real-time to everything the user is doing and everything the back-end is feeding them. Finding the right way to add that behavior in is where things quickly get messy. The basic MVC pattern, while better organization than nothing, still doesn't directly address how to compose complex reactive application behavior. Combined with a lack of test automation that clearly spells out what's supposed to happen at each step, that makes for a buggy application with unintended side-effects.

####Server development can hold back client development (and vice versa).
Client development teams are often separate from server teams. Stakeholders want to be able to try out their designs and iterate on them. But if you have to coordinate with a server team, set up a test server and set up a test database just to get going on the UI, things can slow to a crawl. The gap between communicating ideas and executing on them not only kills velocity but also lets mistakes and misunderstanding go uncorrected until they are painful to fix.

####Enterprise requirements like localization and user privileges are hard to enforce
Enterprise applications come with a number of global requirements, with localization being a key one. You shouldn't have to always reinvent a localization strategy or access rights strategy, but you do if it's not baked-in to the framework. That usually means repetitive hand-coding everywhere. Leaving these key enterprise concerns manual results in gaps in coverage that can be difficult to track down after the fact.

####Bloated disorganized code that's different for every developer
Rich widget libraries are great, but using them in a clean way is challenging. Often the same composition patterns (like master-detail, mixed-type tabs, action enabling/disabling, panel expanding on a trigger, etc.) reappear within any given application. Yet UI component APIs remain focused at a lower level. There are many conflicting ways to develop these rich behaviors, sometimes none of which are clearly superior to the others. That means not only repetitive code that can easily turn into a spaghetti mess, it also means a *different* mess per developer.

####Development is painfully slow
All of this adds up to a painfully slow development cycle in which getting to a high-quality finish for a large application takes much longer than expected, if at all.

###The GluJS solution

GluJS directly addresses the needs of enterprise developers:

####Test-first out-of-the box
GluJS provides a test specification methodology (backed by Jasmine) that captures and turn complex UI behavior provided by stakeholders into actionable code and automated tests. To make sure that the tests correspond to user stories in the real world, GluJS provides a test library that provides full transparent AJAX mocking and dummy data.

####Fast UI cycles even if the server-side is incomplete
Client developers no longer have to wait to get back to the stakeholders. A rich, simple Ajax back-end simulator lets you jump immediately into producing tested UI code. It also lets developers return the client application back to stakeholders for feedback and review -- perhaps before the actual server is even started. Pointing back to an actual server is later a single configuration switch.

####Tight, organized DRY code
GluJS lets you write minimalistic UI-oriented code. Out-of-the box UI patterns like enabling/disabling action buttons based on UI state combined with conventions that automatically wire components together greatly reduces repetitive code, letting the application underneath emerge.

####Enterprise-ready localization and access rights
Localization is not an optional add-on to GluJS, but built-in to the core. As long as you provide the translations in a local file, everything is localized automatically. Plus, GluJS's support for dynamic global control customizations makes it simple to manage global access control patterns like enable/disabling/hiding controls based on user privileges.

####Strong reusable design patterns for distributed teams
Though a great start, the MVC pattern is little more than a pattern for wiring up handlers. The patterns built into GluJS directly address higher-level real-world UI interactions. Since the patterns are so strong, there's a known best way to approach most problems. That helps keep the design of the application unified even with very different developers. And of course that makes for a less-costly, more supportable application.

####Get the most out of best-of-breed HTML 5 graphic libraries
Libraries like Sencha ExtJS provide a rich, clean look and feel, utilities, and Ajax connectivity. GluJS provides the "glue" that brings these pieces together for testable, fast enterprise development.

####Drops in place into existing projects
We realize that as an enterprise developer you are often fitting your application into a larger portal or shell. GluJS has multiple entry points that let you drop your application into a larger one, or write the entire app in GluJS.


*Copyright 2012 Mike Gai. All rights reserved.*