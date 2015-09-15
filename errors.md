# Errors and thier (potential) solution
This is a list of errors that have had me scratching my head because, per usual, the stack contains minified, esoteric info. So what I've tried to do is make a list of the error I was getting and what I eventually did to fix it.

### Unhandled Promise Rejection ReferenceError: Info is not defined

***WHAT?***

Gettin an error that looks a lot like this?

```markdown
Unhandled promise rejection ReferenceError: info is not defined
    at Container.invoke (http://localhost:9000/jspm_packages/github/aurelia/dependency-injection@0.10.0/aurelia-dependency-injection.js:401:30)
    at Array.<anonymous> (http://localhost:9000/jspm_packages/github/aurelia/dependency-injection@0.10.0/aurelia-dependency-injection.js:272:44)
    at Container.get (http://localhost:9000/jspm_packages/github/aurelia/dependency-injection@0.10.0/aurelia-dependency-injection.js:329:24)
    at http://localhost:9000/jspm_packages/github/aurelia/templating@0.15.1/aurelia-templating.js:3685:75
    at f (http://localhost:9000/jspm_packages/npm/core-js@0.9.18/client/shim.min.js:1415:56)
    at http://localhost:9000/jspm_packages/npm/core-js@0.9.18/client/shim.min.js:1423:13
    at b.exports (http://localhost:9000/jspm_packages/npm/core-js@0.9.18/client/shim.min.js:453:24)
    at b.(anonymous function) (http://localhost:9000/jspm_packages/npm/core-js@0.9.18/client/shim.min.js:1625:11)
    at Number.f (http://localhost:9000/jspm_packages/npm/core-js@0.9.18/client/shim.min.js:1596:24)
    at q (http://localhost:9000/jspm_packages/npm/core-js@0.9.18/client/shim.min.js:1600:11)(anonymous function) 
    @ shim.min.js:1444b.exports
    @ shim.min.js:453b.(anonymous function)
    @ shim.min.js:1625f
    @ shim.min.js:1596q
    @ shim.min.js:1600
```

***WHY?***

###### You could be referencing object internals when the object doesn't exist
Happens to me all the time when I try to 'namespace' things inside the `api` object like

```js
this.api.locations = LocationService;
```
but I forgot to declare `api` first! Which totally makes sense because clearly `info is not defined` makes 100% sense and there's no way I could have missed that it actually meant `api is undefined`. Nope. No way to make that mistake.

```js
export class Foo
{
	api = {}; // this is what I always forget
	
	constructor(LocationService)
	{
		this.api.locations = LocationService;
	}
}
```