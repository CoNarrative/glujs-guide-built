##Why GluJS? The extended version

Developing rich web and mobile HTML-based applications in Javascript can be new territory for many enterprise development teams. While there are great libraries like Sencha ExtJS to make a "desktop-like" experience, even ExtJS experts face significant challenges turning the initial "it looks great!" ExtJS wireframes into an actual large working application:

###Existing challenges

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


*Copyright 2012 Mike Gai. All rights reserved.*