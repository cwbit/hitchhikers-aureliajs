# Custom Elements with inner HTML

***WHAT?***

Sometimes when you use a custom element you need to give it internal HTML and then use that inside the custom element template. You could do this with a bound variable but that sucks if you don't actually need to.

e.g. if you want to be able to do `<sexy-header>Oh Yeah<sexy-header>` there's no reason to mess with tiny JS classes and bound variables...

***HOW***

first, wrap your content inside the custom-element tags

```html
	<!-- any view html file -->
    <require from="sexy-header.html"></require>
    <sexy-header>Print me, please!</sexy-header>
```

then, in your custom element, use the `<content></content>` tags

```html
<!-- inside sexy-header.html -->
<template>
	<div class="sexy header">
		<content></content><!-- magic 'bout to happen -->
	</div>
</template>
```

Which would spit out ...

```html
	<div class="sexy header">Print me, Please!</div>
```

NICE!