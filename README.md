# LiveView & AlpineJS - A Guide to Successful Integration

This document lists some of the best practices of integrating the AlpineJS framework with Phoenix LiveView.

There may be some redundancies with the Alpine or LiveView official documentation as the point is to have it all in
one single place here.

The document is intended to evolve over time as the needs arise.

## Installation

Go to the `assets` directory in your project (or your web app if an Umbrella project) and run the code below:

```
npm install alpinejs
```

The following are the required modifications of the LiveView `app.js` file.

##### Imports

```javascript
import Alpine from 'alpinejs'

// Alpine plugins you may need come here (optional)
import intersect from '@alpinejs/intersect'
import TimeAgo from '@marcreichel/alpine-timeago'
..
```

##### JS functions made visible to Alpine code (optional)

```javascript
window.doSomething = function( arg) {
  ..
};

window.doSomethingElse = function( arg1, arg2) {
  ..
};

..
```

##### Alpine and Alpine plugin initialization

```javascript
window.Alpine = Alpine;

// Any Alpine plugins you may have imported above (optional)
Alpine.plugin( intersect);
Alpine.plugin( TimeAgo);
..

// Alpine initialization
Alpine.start();
```

##### Alpine related hooks (optional)

```javascript
// Any hooks relating to Alpine go here, eg:
const InitAlpine = {
  mounted() {
    Alpine.initTree( this.el);
  }
};

let hooks = {
  InitAlpine: InitAlpine
  // other, Alpine unrelated hooks you may have
  .. 
};
```

##### Instantiating the liveSocket

```javascript
let csrfToken = document.querySelector("meta[name='csrf-token']").getAttribute("content");
let liveSocket = new LiveSocket( "/live", Socket, {
  params: { _csrf_token: csrfToken},
  hooks: hooks,
  dom: {
    onBeforeElUpdated( from, to) {
      if ( from._x_dataStack) {
        window.Alpine.clone( from, to);
      }
    }
  }
});

.. 

```

## Dos and Don'ts

### The `x-init` code

Any and all AlpineJS code leveraging the LiveView-Alpine interoperability should be placed in the respective elements'
`x-init` attribute. 

It is the code in `x-init` that's triggered each time any of the interpolated LiveView assigns is updated. By relying
on this behavior the Elixir code can achieve a complete control over the Alpine JS code execution. If used properly
it is possible to building a complex UX without having to write a single LiveView hook (except maybe the one shown in
the [Installation](#alpine-related-hooks-optional)).     

Ex:

```
<div
  id={@id}
  x-data="{ someState: '' }"
  x-init={"
    someState = #{ @some_data};
    somethingChanged = #{ @something_changed};
    
    if( someState && somethingChanged) {
      ..
    }
  "}
>
  ..
</div>
```
 
In the example above `someState` becomes a part of the Alpine managed state (by virtue of being defined in the `x-data`)
and is available as such to all Alpine attributes in the element and its descendants, while `somethingChanged` is just
a variable visible only in the `x-init` code block.
 
Given that `someState` and `somethingChanged` are each set to a LiveView assign, the `x-init` code will be executed the
first time the element is rendered and each time any of the two assigns changes.  

If the desired behavior is to have the `x-init` code invoked without any (JS-wise semantic) change to either
`@some_data` or  `@something_changed`, then at least one of the two must be versioned or assigned a value keeping the
same semantics relative to the Alpine code, e.g. an ever incrementing integer starting from `1` (in JS `0` would be
treated the same as `false` in the `if` statement above). 

In a more complex case, an actual versioning should be used, e.g.:

```
x-init={"
  { someState} = { someState: '#{ @some_data.value}', version: #{ @some_data.version}};
  .. 
"}
```

Thus, even if `@some_data.value` is not changed, the update of the version part of the assign will make LV render the
element and have the `x-init` reevaluated.

### The code in `x-data` and other Alpine attributes

Here, the rule is very simple - either interpolate only the assigns that are constants in their nature or don't
interpolate any assigns at all.

### When the element id is a must

An Alpine-leveraging element must have a (unique) `id` if there are any (LiveView-wise) conditionally rendered elements
preceding it. 

Alpine relies on the element `id` to unambiguously identify which element state is being referenced. When there's no
`id`, Alpine relies on the nearest ancestor element that has an `id` and the relative position of the element in
question in the DOM structure. Obviously, if there are any structural changes between the element and the id-ed ancestor
the result will be an unpredictable and buggy Alpine behavior.

### Prepending items to a stream in LV v0.19.x 

```elixir
stream_insert( socket, :my_items, item, at: 0)
```

As of LV v0.19.x, prepending an item to or inserting it at a certain position in a stream no longer has the item
element's Alpine code initialized (neither `x-data` nor `x-init` get evaluated on mount). 

Obviously, this results in all state-dependent Alpine JS code in such an element crashing.

To remedy this, it is required to define the hook provided in the
[installation guide](#alpine-related-hooks-optional), and then use it in all items that get prepended or inserted
`:at` a specific position in the stream.

Ex:

```
<div 
  id={@id} phx-hook={"InitAlpine"} 
  .. 
>
  ..
</div>
``` 

If the item is getting prepended/inserted `:at` conditionally (and appended otherwise), then the hook use can be
conditioned too, e.g.:

```
<div 
  id={@id} phx-hook={@prepended? && "InitAlpine"} 
  .. 
>
  ..
</div>
``` 
