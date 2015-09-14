# Event Aggregating Inspection Window

This discovery was so exciting that I immediately had to sit down and write a tutorial on it.. we're gonna make an "Inspection" pane, like in Photoshop or something, that loads details based on what you clicked and requires *only* that the clicked object communicate with it thru a regulated, decoupled message class. If everything works as planned, our Inspection window will be able to display *anything* we want, without any one element knowing more than it should.

>Or, said differently, we can force a loosely-coupled DOM object to dynamically load a custom element based on any event - like a user click

***WHY?***
OK so maybe you're asking WHY would you ever want to do this. Good Question. This tutorial was actually spawned out of a very specific use-case. I wanted to be able to click stuff on a web-game I was making and have a sidebar show more info about it. But I also didn't want the sidebar to have to know what it was about to render. That'd mean that my sidebar and the various points on my app were 'tightly coupled' - if I changed one part I'd probably have to change a bunch of others.

Additionally if I ever wanted to display information in more than one place I'd have to put `if..then..` or `switch` cases all over the place - which is **BAD BAD BAD**. Actually Facebook ran into that problem with thier chat application - they wanted multiple things to be able to respond to what was going on - and had such a bad time with it that they ended up standardizing a whole new (to them) design paradigm of how to make a more event-driven approach to design.

So while this tutorial is semi-specific (inspector window being driven by click events), ==what this tutorial really teaches you is publish and subscribe to events in your app and have those events be able to dynamically change portions of your UI while preserving full Aurelia functionality== (event delegation, data-bindings, etc - which would all be lost with regular string or DOM injected elements).

To illustrate this, let's consider the following example.

### The Gist of what we want to do

Consider the following (crude) diagram. When clicked, each of the buttons should load a special custom element into the Inspection Window .. but without knowing barely anything about how it's done, and **absolutely nothing** about who/what/where/how things are being implemented.

```
			           THE INSPECTION
	 MAIN WINDOW           WINDOW
 ---------------------    ---------
|                     |  |   FOO   |
|     [I'M A FOO]     |  |         |
|                     |  | [TAKE] |
|     [I'M A BAR]     |  | [EXIT]  |
|                     |  |         |
 ---------------------    ---------
```

Sounds.. actually, it's easy!

Here's the gist of what we're going to do. 

1. We're going to create a strongly-typed `InspectMessage` to be used for communication with the Inspection Window
2. We're going to make a button click event create a new instance of the `InspectMessage` and send it to Aurelia's `EventAggregator`
3. The Inspection Window will subscribe to the `InspectMessage` using the `EventAggregator`
4. When the Inspection Window gets an `InspectMessage` event it parses data out that the special `<compose>` tag will bind to; causing our dynamic elements to load on-demand

### Creating our basic `App`

First, lets create our basic `app.html`. This just creates two containers, one with buttons, and one based on the `inspector` view/model that we'll build next.

```html
<!-- app.html -->
<div class='container'>
	<!-- load our custom element files -->
	<require from="inspector"></require>
	
	<!-- here's our clickables -->
	<div class='clickable stuff'>
		<button click.delegate="inspectMe($event, {type:foo, .. }) />
		<button click.delegate="inspectMe($event, {type:bar, progress:42, .. }) />
	</div>
	
	<!-- and here's our custom inspector element (loaded above) -->
	<inspector></inspector>
</div>
```

The view model for our simple app will just have an `inspectMe()` function and will use the `EventAggregator` and a custom `InspectMessage` class that we'll create later.

```javascript
// app.js
import {inject} from 'aurelia-framework';
import {EventAggregator} from 'aurelia-event-aggregator';
import {InspectMessage} from './messages/inspect-message.js';

@inject(EventAggregator)
export class App
{
	// gets the injected objects in order
	constructor(ea)
	{
		this.ea = ea; //store the EventAggregator instance
	}
	
	// takes the click event and some object data and sends the message to subscribed listeners
	inspectMe(event, data)
	{
		this.ea.publish(new InspectMessage('custom-' + data.type + '-element', data));
	}
}
```

That's it for the main App. It prints out some buttons which call `inspectMe` when clicked. The click event then takes the data passed to it and creates a new instance of `InspectMessage` (which we'll create shortly) which takes an element name and some data. This `InspectMessage` event is then sent to anyone who's subscribed to the `InspectMessage` event channel. 

>Notice, the **element** name used in our `new InspectMessage(..)` is based on the `type` value passed in by our button click.

### Creating our 'Inspector Panel'

Next, let's create our simple inspector view/model with the magical `<compose>` tag.

```html
<!-- inspector.html -->
<template>
	<compose
		view-model.bind="inspector.viewModel"
		model.bind="inspector.modelData">
	</compose>
</template>
```

OK. So what is that `<compose>` tag? Basically it's a special Aurelia tag that allows you to dynamically load view/view-models. We're using 2 of its attributes:

* `view-model=".."` - tells `compose` what custom element to load
* `model="{..}"` - this misleadingly named attribute allows `<compose>` to pass _data_ into the model being called. It does _not_ specify which model - that's what the `view-model` attribute is for.

>Note, if the view-model class has an `activate()` function, compose will give it `model` as the first param - e.g. `activate (model) { .. }` where `model` is just the data object we passed from `<compose model.bind="{..}">`. This is important for what we're about to do.
 
Ok, let's finish our `inspector` class and make it use the `EventAggregator` and listen for `InspectMessage`.

```javascript
// inspector.js
import {inject} from 'aurelia-framework';
import {EventAggregator} from 'aurelia-event-aggregator';
import {InspectMessage} from './messages/inspect-message.js';

@inject(EventAggregator)
export class Inspector
{
	inspector = {
		viewModel = '',
		modelData = {}
	}
	
	constructor(ea)
	{
		// store the EventAggregator
		this.ea = ea;
		
		// watch for an InspectMessage and, when found, 
		// parse out viewModel and modelData and save them 
		// so our `compose` tag picks them up with its bindings
		this.ea.subscribe(InspectMessage, (msg) => {
			this.inspector.viewModel = msg.viewModel;
			this.inspector.modelData = msg.modelData;
		})
	}	
}
```

That's it. Our `Inspector` will now watch for `InspectMessage` events, take the `viewModel` and `modelData` from the message, and store the variables in it's own internal `inspector` object (to keep things clean).

Since the `<compose>` tag is data-bound to `viewModel` and `modelData` it will automatically update itself whenever these two variables change in `inspector.js`.

This means that when we click a button it will emit an event (and some data) that will be picked up by our Inspector and used to dynamically render a complete custom element.

### Creating the `InspectMessage`

Let's make the quick `InspectMessage` class that ferrys the data between publisher and subscriber, and then round this tutorial out with two dynmically loaded elements.

```javascript
// ./messages/inspect-message.js
export class InspectMessage
{
    constructor(viewModelToLoad, andItsModelData){
        this.viewModel = viewModelToLoad;
        this.modelData = andItsModelData;
    }
}
```

Simple. Requires two constructor parameters, `viewModelToLoad` and `andItsModelData` and stores them for Subscribers to use.

For an example on how to publish or subscribe to this `InspectMessage` go re-read the `App` and `Inspector` js classes we made earlier.

### Creating some arbitrary elements to load dynamically

Next let's create two tiny elements that print out custom code. These are just examples to demonstrate proof that we are actually loading things dynamically through our event-aggregated-compose-loading system. These really could be anything.

For our first dummy element we'll just print out some text and two buttons that do nothing.

```javascript
// custom-foo-element.js
export class Foo
{
	activate(modelDataFromCompose)
	{
		this.data = modelDataFromCompose;
	}
}
```

```html
<!-- custom-foo-element.html -->
<template>
	<p>I'm a Foo</p>
	<button>Take It</button>
	<button>Leave It</button>
</template>
```

And for our second custom element we'll create a progress bar (psuedo-code really, it doesnt actually make one) that uses information passed to it thru the `InspectMessage` event. 

```javascript
// custom-bar-element.js
export class Bar
{
	activate(modelDataFromCompose)
	{
		this.progress = modelDataFromCompose.progress;
	}
}
```

```html
<!-- custom-bar-element.html -->
<template>
	<p>I'm a Bar</p>
	<div class="bar" width="100px">
		<div class="progress" width.bind="progress">&nbsp;</div>
	</div>
</template>
```

### Putting it all together

Ok, it's getting late, so let's quickly tie all that together. Because we're actually **DONE!**

What we have are TWO buttons that, when clicked, submit a message thru the special `InspectMessage` class along with some data and the name of thier custom element: either `custom-foo-element` or `custom-bar-element`. Then, because our `inspector` view/view-model is loaded, it registers itself to listen for any `InspectMessage`. The inspectors `<compose>` tag is data-bound to the empty `inspector` object and, on the initial page load, draws an empty DOM element.

Then, when one of the buttons is clicked, the `InspectMessage` is published by the click event and picked up by the `Inspector` class (because it _subscribed_ to the `InspectMessage`). This event is then unpacked and the name of the view model to draw and the data to pass to it are pulled from the message and stored in the `inspector` object. This change to the `inspector` object triggers an update of the observing (data-bound) `<compose>` tag. `compose` loads the view/view-model of the element indicated by `inspector.viewModel` and passes it the data stored in `inspector.modelData`.

When either of the templates is `activate()`ed it's also given the result of `inspector.modelData` by the `compose` tag. It then parses out this data into whatever form it wants and loads itself.

As a result, clicking either button will dynamically alter the DOM object of the Inspector, causing it to load and destroy (or activate and deactivate) Aurelia elements on demand!