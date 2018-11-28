- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Tracked Properties

## Summary

Tracked properties introduce a simpler and more ergonomic system for tracking
state change in Ember applications. By taking advantage of new JavaScript
features, tracked properties allow Ember to reduce its API surface area while
producing code that is both more intuitive and less error-prone.

This simple example shows a `Person` class with three tracked properties:

```js
export default class Person {
  @tracked firstName = 'Chad';
  @tracked lastName = 'Hietala';

  @tracked get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

### A Note on Decorator Stability

Decorator support in Ember is crucial for the release of Ember Octane. Native
classes are not usable without decorators, and they are a major part of the next
edition. That said, decorators are still a stage 2 proposal in TC39, and merging
support for them before stage 3 represents a risk.

This RFC is being made now so that we can get early discussion out of the way,
and agree on the general parameters of the `@tracked` decorator in Ember.
Ideally, there will be no major changes before decorators move to stage 3, and
we'll be able to merge this RFC shortly after it is announced.

Alternatively, if decorators do not move forward in TC39 in the next few months,
we may need to consider making an exception for tracked properties, or scoping
down Ember Octane.

## Terminology

Because of the occasional overlap in terminology when discussing similar
features, this document uses the following language consistently:

- A **getter** is a JavaScript feature that executes a function to determine the
  value of a property. The function is executed every time the property is
  accessed.
- A **computed property** is a property on an Ember object whose value is
  produced by executing a function. That value is cached until one of computed
  property's dependencies changes.
- A **tracked property** refers to any property that has been instrumented with
  `@tracked`, either a _tracked getter_ or a _tracked simple property_.
- A **tracked getter** is a JavaScript getter that has been wrapped using the
  tracked decorator
- A **tracked simple property** is a regular, non-getter property that has been
  wrapped using the tracked decorator.
- The **legacy programming model** refers to the traditional Ember programming
  model. It includes _legacy classes_, _computed properties_, _event listeners_,
  _observers_, _property notifications_, and _curly components_, and more
  generally refers to features that will eventually be removed from Ember.
- **Native classes** are classes defined using the Javascript `class` keyword.
- **Legacy classes** are classes defined by subclassing from `EmberObject` using
  the static `extend` method.

## Motivation

Tracked properties are designed to be simpler to learn, simpler to write, and
simpler to maintain than today's computed properties. In addition to clearer
code, tracked properties eliminate the most common sources of bugs and mental
model confusion in computed properties today.

### Leverage Existing JavaScript Knowledge

Ember's computed properties provide functionality that overlaps with native
JavaScript getters and setters. Because native getters don't provide Ember with
the information it needs to track changes, it's not possible to use them
reliably in templates or in other computed properties.

New learners have to "unlearn" native getters, replacing them with Ember's
computed property system. Unfortunately, this knowledge is not portable to other
applications that don't use Ember that developers may work on in the future.

Tracked properties are as thin a layer as possible on top of native JavaScript.
Tracked properties look like normal properties because they _are_ normal
properties.

Because there is no special syntax for retrieving a tracked property, any
JavaScript syntax that feels like it should work does work:

```js
// Dot notation
const fullName = person.fullName;
// Destructuring
const { fullName } = person;
// Bracket notation for computed property names
const fullName = person['fullName'];
```

Similarly, syntax for changing properties works just as well:

```js
// Simple assignment
this.firstName = 'Yehuda';
// Addition assignment (+=)
this.lastName += 'Katz';
// Increment operator
this.age++;
```

This compares favorably with APIs from other libraries, which becomes more
verbose than necessary when JavaScript syntax isn't available:

```js
this.setState({
  age: this.state.age + 1,
});
```

```js
this.setState({
  lastName: this.state.lastName + "Katz";
})
```

### Avoiding Dependency Hell

Currently, Ember requires developers to manually enumerate a computed property's
dependent keys: the list of _other_ properties that _this_ computed property
depends on. Whenever one of the listed properties changes, the computed
property's cache is cleared and any listeners are notified that the computed
property has changed.

In this example, `'firstName'` and `'lastName'` are the dependent keys of the
`fullName` computed property:

```js
import EmberObject, { computed } from '@ember/object';

const Person = EmberObject.extend({
  fullName: computed('firstName', 'lastName', function() {
    return `${this.firstName} ${this.lastName}`;
  }),
});
```

While this system typically works well, it comes with its share of drawbacks.

First, it's annoying to have to type every property twice: once as a string as a
dependent key, and again as a property lookup inside the function. While
explicit APIs can often lead to clearer code, this verbosity often obfuscates
the intent of the property. People understand intuitively that they are typing
out dependent keys to help _Ember_, not other programmers.

Second, people tell us that this syntax is not very intuitive. You have to read
the Ember documentation at least once to understand what is happening in this
example.

It's also not clear what syntax goes inside the dependent key string. In this
simple example it's a property name, but nested dependencies become a property
path, like `'person.firstName'`. (Good luck writing a computed property that
depends on a property with a period in the name.)

You might form the mental model that a JavaScript expression goes inside the
string—until you encounter the `{firstName,lastName}` expansion syntax or the
magic `@each` syntax for array dependencies.

The truth is that dependent key strings are made up of an unintuitive,
unfamiliar microsyntax that you just have to memorize if you want to use Ember
well.

Lastly, it's easy for dependent keys to fall out of sync with the
implementation, leading to difficult-to-detect, difficult-to-troubleshoot bugs.

For example, imagine a new member on our team is assigned a bug where a user's
middle name is not appearing in their profile. Our intrepid developer finds the
problem, and updates `fullName` to include the middle name:

```js
import EmberObject, { computed } from '@ember/object';

const Person = EmberObject.extend({
  fullName: computed('firstName', 'lastName', function() {
    return `${this.firstName} ${this.middleName} ${this.lastName}`;
  }),
});
```

They test their change and it seems to work. Unfortunately, they've just
introduced a subtle bug. If the user's `middleName` were to change, `fullName`
wouldn't update! Maybe this will get caught in a code review, given how simple
the computed property is, but noticing missing dependencies is a challenge even
for experienced Ember developers when the computed property gets more
complicated.

Tracked properties have a feature called _autotrack_, where dependencies are
automatically detected as they are used. This means that as long as all
dependencies are marked as tracked, they will automatically be detected:

```js
import { tracked } from '@ember/object';

class Person {
  @tracked firstName = 'Tom';
  @tracked lastName = 'Dale';

  @tracked
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

This also allows us to opt out of tracking entirely, like if we know for
instance that a given property is constant and will never change. In general,
the idea is that _mutable_ properties should be marked as tracked, and
_immutable_ properties should not.

### Reducing Memory Consumption

By default, computed properties cache their values. This is great when a
computed property has to perform expensive work to produce its value, and that
value gets used over and over again.

But checking, populating, and invalidating this cache comes with its own
overhead. Modern JavaScript VMs can produce highly optimized code, and in many
cases the overhead of caching is greater than the cost of simply recomputing the
value.

Worse, cached computed property values cannot be freed by the garbage collector
until the entire object is freed. Many computed properties are accessed only
once, but because they cache by default, they take up valuable space on the heap
for no benefit.

For example, imagine this component that checks whether the `files` property is
supported in input elements:

```js
import Component from '@ember/component';
import { computed } from '@ember/object';

export default Component.extend({
  inputElement: computed(function() {
    return document.createElement('input');
  }),

  supportsFiles: computed('inputElement', function() {
    return 'files' in this.inputElement;
  }),

  didInsertElement() {
    if (this.supportsFiles) {
      // do something
    } else {
      // do something else
    }
  },
});
```

This component would create and retain an `HTMLInputElement` DOM node for the
lifetime of the component, even though all we really want to cache is the
Boolean value of whether the browser supports the `files` attribute.

Particularly on mobile devices, where RAM is limited and often slow, we should
be more conservative about our memory consumption. Tracked properties switch
from an opt-out caching model to opt-in, allowing developers to err on the side
of reduced memory usage, but easily enabling caching (a.k.a. memoization) if a
property shows up as a bottleneck during profiling.

## Detailed Design

This RFC proposes two new exports:

1. The `tracked` decorator function, used to mark properties and getters as
   tracked
2. The `notifyPropertyChange` function, which is currently available on the
   Ember global, but hasn't been made available from module exports.

```ts
const tracked: PropertyDecorator;

function notifyPropertyChange(obj: any, property: string): void;
```

Both of these new functions will be exported from `@ember/object`.

Revisiting our example from earlier, `@tracked` can be used on native class
fields and getters/setters:

```js
import EmberObject, { tracked } from '@ember/object';

class Person {
  @tracked firstName = 'Tom';
  @tracked lastName = 'Dale';

  @tracked
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

### Getting Tracked Properties

Tracked properties can be accessed using standard Javascript syntax. From the
user's point of view, there is nothing special about them. This should continue
to work in the future, even if new methods are added for accessing properties,
because tracked properties use native getters under the hood.

```js
let person = new Person();

// Dot notation
const fullName = person.fullName;
// Destructuring
const { fullName } = person;
// Bracket notation for computed property names
const fullName = person['fullName'];
```

### Setting Tracked Properties

Tracked properties can be set using standard Javascript syntax. They use native
setters under the hood, meaning that there is no need for using a setter method
like `set`.

```js
let person = new Person();

// Simple assignment
person.firstName = 'Jen';
// Addition assignment (+=)
person.lastName += 'Weber';
// Increment operator
person.age++;
```

### Autotracking

Tracked properties do not need to specify their dependencies. Under the hood,
this works by utilizing an _autotrack stack_. This stack is a bit of global
state which tracked getters can access. As tracked getters and properties are
accessed, they push themselves onto the stack, and once they have finished
running, the stack contains the full list of all the tracked properties that
were accessed while it was running.

In our first example, with the `Person` class, we can see this in action:

```js
import EmberObject, { tracked } from '@ember/object';

class Person {
  @tracked firstName = 'Tom';
  @tracked lastName = 'Dale';

  @tracked
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}

let person = new Person();
```

When we create a new instance of `Person`, `fullName` has no knowledge of
`firstName` or `lastName`. If we set either of those values now, it won't know
that anything has changed:

```js
// Dirties the `firstName` property, but because `fullName` has not been
// accessed yet nothing happens to it.
person.firstName = 'Rob';
```

This is ok though, because _nothing_ has accessed `fullName` yet. There is no
state to invalidate anywhere else. Now, let's say we access `fullName`, and then
update the field again:

```js
person.fullName; // 'Rob Dale'

person.lastName = 'Jackson';
```

Now `fullName` knows which tracked properties were accessed when it was run,
and setting `lastName` has invalidated `fullName`.

> **NOTE:** This does _not_ invalidate a cache like in computed properties. Even
> if `firstName` and `lastName` were untracked, the tracked getter would still
> return the correct value on subsequent accesses, because `@tracked` does _not_
> cache values. The validation and invalidation is pure meta data that is only
> accessible by the Glimmer VM.
>
> Internally, Glimmer checks to see if a value has updated _before calling the
> getter_. If it hasn't, then Glimmer does not rerender the related section of
> the DOM. This is effectively an automatic `shouldComponentUpdate` from React.
>
> To prevent inconsistency, during development time, tracked properties will
> keep a cache of their previous value to compare when they are activated and
> ensure that it hasn't changed without invalidation. This will prevent improper
> usage of tracked properties _outside_ of Glimmer's change tracking.

### Manual Invalidation

In user code, the idea that all mutable properties should be marked as tracked
and that all other properties are effectively immutable works well in isolation.
However, there are cases where users will want to work with code they do _not_
control, such as external library code.

Consider the following example. We have a `simple-timer` library that we've
imported from NPM, and we're trying to wrap it with a `TimerComponent` that
uses it to keep track of how much time has passed:

```js
// simple-timer/index.js
export default class Timer {
  seconds = 0;
  minutes = 0;
  hours = 0;

  listeners = [];

  constructor() {
    setInterval(() => {
      this.seconds++;
      this.minutes = Math.floor(this.seconds / 60);
      this.hours = Math.floor(this.minutes / 60);
      this.notifyTick();
    }, 1000);
  }

  notifyTick() {
    for (let listener of this.listeners) {
      listener(this.seconds);
    }
  }

  onTick(listener) {
    this.listeners.push(listener);
  }
}
```

```js
import Timer from 'simple-timer';
import Component from '@ember/component';
import { tracked } from '@ember/object';

export default class TimerComponent extends Component {
  @tracked timer = new Timer();

  @tracked
  get currentSeconds() {
    return this.timer.seconds;
  }

  @tracked
  get currentMinutes() {
    return this.timer.minutes;
  }
}
```

Even though we've marked the `timer` property as tracked, the `timer.seconds`
property is untracked, and _it_ is the field that is updated. We can solve this
problem by using the timer library's `onTick` event handler to re-set the field,
invalidating it:

```js
export default class TimerComponent extends Component {
  @tracked timer = new Timer();

  constructor() {
    this.timer.onTick(() => {
      // invalidate the timer field.
      this.timer = this.timer;
    });
  }

  @tracked
  get currentSeconds() {
    return this.timer.seconds;
  }

  @tracked
  get currentMinutes() {
    return this.timer.minutes;
  }
}
```

This invalidates our properties whenever the timer updates. This is a simple
technique which can be used in many cases with complex classes, POJOs, and
arrays, but you may have noticed that we are invalidating `currentMinutes` more
often than we should. This is inefficient, and could be made better by directly
using `notifyPropertyChange`:

```js
export default class TimerComponent extends Component {
  timer = new Timer();

  @tracked seconds;
  @tracked minutes;

  constructor() {
    this.timer.onTick(() => {
      // invalidate the seconds value
      notifyPropertyChange(this, 'currentSeconds');

      if (this.timer.seconds % 60 === 0) {
        // invalidate the seconds value
        notifyPropertyChange(this, 'currentMinutes');
      }
    });
  }

  @tracked
  get currentSeconds() {
    return this.timer.seconds;
  }

  @tracked
  get currentMinutes() {
    return this.timer.minutes;
  }
}
```

This API is intended to be used primarily by _addon and library_ developers who
want to interoperate with non-Ember libraries, and is not very different from
equivalent interop layer code which uses computed properties and property
notifications:

```js
export default class TimerComponent extends Component {
  timer = new Timer();

  constructor() {
    this.timer.onTick(() => {
      // invalidate the seconds value
      notifyPropertyChange(this.timer, 'seconds');

      if (this.timer.seconds % 60 === 0) {
        // invalidate the seconds value
        notifyPropertyChange(this.timer, 'minutes');
      }
    });
  }

  @computed('timer.seconds')
  get currentSeconds() {
    return this.timer.seconds;
  }

  @computed('timer.minutes')
  get currentMinutes() {
    return this.timer.minutes;
  }
}
```

### Interop with the Legacy Programming Model

Tracked properties represent a paradigm shift. They are a completely new system,
fully independent of the legacy programming model, which will allow us to
eventually deprecate and remove the old system in favor of the new one based on
modern Javascript features and design.

However, that won't happen over night. Existing apps, libraries, and addons will
have to be gradually rewritten over time, and experience tells us that these
sort of transitions take a while to settle in the community. The last major
shift of this scale was the framework's removal of Views in lieu of Components,
which took a full major version and some change to finish.

To ease this process and enable gradual adoption, tracked properties will be
able to interoperate with the most commonly used features of the legacy model:

- Legacy classes
- Computed properties
- `get`/`set` and property notifications

Tracked properties will _not_ interoperate with observers, which are strictly
within the old paradigm.

#### Legacy Classes

The `tracked` decorator function will be usable in legacy classes, similar to
`computed`:

```js
import EmberObject, { tracked } from '@ember/object';

const Person = EmberObject.extend({
  firstName: tracked({ value: 'Tom' }),
  lastName: tracked({ value: 'Dale' }),

  fullName: tracked({
    get() {
      return `${this.firstName} ${this.lastName}`;
    },
  }),
});
```

This form will _not_ be allowed on native classes, and will hard error if it is
attempted. Additionally, default values will be defined on the _prototype_ to
maintain consistency with the legacy object model.

This will allow existing libraries to transition incrementally, and add tracked
support minimally where necessary. This also brings the _benefits_ of tracked
to legacy classes, including the ability to drop usage of `set`:

```js
// before
let person = Person.create();
person.set('firstName', 'Tom');
person.set('lastName', 'Dale');

// after
let person = Person.create();
person.firstName = 'Tom';
person.lastName = 'Dale';
```

Ember's `set` function is nowhere to be seen!

#### Computed Properties

Computed properties will interoperate with tracked properties in both
directions:

- Accessing a computed property from a tracked property will add the computed
  property to its list of depedencies. Whenever the computed property is
  invalidated (i.e. because it or one of its dependencies is updated), the
  tracked property will be invalidated as well.

  ```js
  import { set, tracked } from '@ember/object';
  import { alias } from '@ember/object/computed';

  class Person {
    @tracked firstName;
    @tracked lastName;

    @alias('title') prefix;

    @tracked
    get fullName() {
      return `${this.prefix} ${this.firstName} ${this.lastName}`;
    }
  }

  let person = new Person();

  person.firstName = 'Tom';
  person.lastName = 'Dale';

  set(person, 'title', 'Mr.');

  person.fullName; // 'Mr. Tom Dale'
  ```

- Accessing a tracked property from a computed property will _also_
  automatically add the tracked property to the list of its dependencies. In
  this way, users will be able to gradually add tracked properties and
  simultaneously reap the benefits of not having to use `set` with computeds,
  and not having to specify dependent keys.

  ```js
  import { tracked, computed } from '@ember/object';

  class Person {
    firstName;
    lastName;

    @tracked middleName;

    @computed('firstName', 'lastName')
    get fullName() {
      return `${this.firstName} ${this.middleName} ${this.lastName}`;
    }
  }

  let person = new Person();

  set(person, 'firstName', 'Tom');
  set(person, 'lastName', 'Dale');

  person.middleName = 'Tomster';

  person.fullName; // 'Tom Tomster Dale'
  ```

#### `get` and `set`

It is common in the legacy model to set and consume plain object properties
which are not computed properties, or in any other way special. Ember's `get`
and `set` functions historically allowed this by giving us the ability to
intercept all property changes and watch for mutations.

This presents a problem for tracked properties, particularly because of the
recent change in Ember to enable native Javascript getters to replace `get`.
This change means that we have no way to intercept `get`, and consequently no
way for tracked properties to know whether or not a plain property will later
be updated with `set`.

To demonstrate this case, consider the following service and component:

```js
const Config = Service.extend({
  polling: {
    shouldPoll: false,
    pollInterval: -1,
  },

  init() {
    this._super(...arguments);

    fetch('config/api/url')
      .then(r => r.json())
      .then(polling => set(this, 'polling', polling));
  },
});
```
```js
class SomeComponent extends Component {
  @service config;

  @tracked
  get pollInterval() {
    let { shouldPoll, pollInterval } = this.config.polling;

    return shouldPoll ? pollInterval : -1;
  }
}
```

Let's walk through the flow here:

1. The `SomeComponent` component is rendered for the first time, instantiating
   the `Config` service (assuming this the first time it has ever been
   accessed). The service's init hook kicks off an async request to get the
   configuration from a remote URl.
2. The tracked `pollInterval` property first accesses the service injection,
   which is a computed property. The property is detected and added to the
   tracked stack.
3. We then access the plain, undecorated `polling` object. Because it is
   is not tracked and not a computed property, tracked does not know that it
   could update in the future.
4. Sometime later, the async request returns with the configuration object. We
   set it on the service, but because our tracked getter did not know this
   property would update, it does not invalidate.

In order to prevent this from happening, user's will have to use `get` when
accessing any values which may be set with `set`, and are not computed
properties.

```js
class SomeComponent extends Component {
  @service config;

  @tracked
  get pollInterval() {
    let shouldPoll = get(this, 'config.polling.shouldPoll');
    let pollInterval = get(this, 'config.polling.pollInterval');

    return shouldPoll ? pollInterval : -1;
  }
}
```

The reverse, however, is not true - computed properties will be able to add
tracked properties, and listen to dependencies explicitly. In some cases, this
may be preferable, though tracked getter should be the conventional standard
with the long term goal of removing all explicit dependencies.

## How we teach this

There are three different aspects of tracked properties which need to be
considered for the learning story:

1. **General usage.** Which properties should I mark as tracked? How do I
   consume them? How do I trigger changes?
2. **Interop with legacy systems.** How do I safely consume tracked properties
   from legacy classes and computeds? How do I safely consume legacy APIs from
   tracked properties?
3. **Interop with non-Ember systems.** How do I tell my app that something has
   changed in MobX objects, RxJS objects, Redux, etc.

### General Usage

The mental model with tracked properties is that anything _mutable_ should be
tracked. If a value will ever change, it should have the `@tracked` decorator
attached to it.

After that, usage should be "Just Javascript". You can safely access values
using any syntax you like, including desctructuring, and you can update values
using standard assignments.

```js
// Dot notation
const fullName = person.fullName;
// Destructuring
const { fullName } = person;
// Bracket notation for computed property names
const fullName = person['fullName'];

// Simple assignment
this.firstName = 'Yehuda';
// Addition assignment (+=)
this.lastName += 'Katz';
// Increment operator
this.age++;
```

#### Triggering Updates on Complex Objects

There may be cases where users want to update values in complex, untracked
objects such as arrays or POJOs. `@tracked` will only be usable with class
syntax at first, and while it may make sense to formalize these objects into
tracked classes in some cases, this will not always be the case.

To do this, users can re-set a tracked value directly after its inner values
have been updated.

```js
class SomeComponent extends Component {
  @tracked items = [];

  @action
  pushItem(item) {
    let { items } = this;

    items.push(item);

    this.items = items;
  }
}
```

This may seem a bit strange at first, but it allows users to mentally scope
off a tree of objects. They manipulate internals as they see fit, and the only
operation they need to do to update state is set the nearest tracked property.

### Interop with Legacy Systems

There are two cases that we need to consider when teaching interoperability:

1. Tracked getters accessing non-tracked properties and computeds
2. Computed getters accessing tracked properties

In the first case, the general rule of thumb is to use `get` if you want to be
100% safe. In cases where you are certain that the values you are accessing are
tracked, computeds, or immutable, you can safely standard access syntax.

In the second case, no additional changes need to be made when using tracked
properties. They can be accessed as normal, and will be automatically added to
the computed's dependencies. There is no need to use `get`, and you can use
standard assignments when updating them.

### Interop with Non-Ember Systems

The `trackedNotifier` and `notifyObjectChange` functions provide a base of
functionality for wrapping external libraries. However, they are simple
primitives and it may be difficult for users to use them effectively. We should
add a guide which specifically demonstrates their usage by wrapping a common,
simple external library such as `moment.js`. This will demonstrate its usage
concretely, and establish best practices.

## Drawbacks

Like any technical design, tracked properties must make tradeoffs to balance
performance, simplicity, and usability. Tracked properties make a different set
of tradeoffs than today's computed properties.

This means tracked properties come with edge cases or "gotchas" that don't exist
in computed properties. When evaluating the following drawbacks, please consider
the two features in their totality, including computed property gotchas you have
learned to work around.

In particular, please try to compensate for [familiarity ][familiarity] and
[loss aversion][loss-aversion] biases. Before you form a strong opinion, [give
it five minutes][5-minutes].

[familiarity]: https://en.wikipedia.org/wiki/Familiarity_heuristic
[loss-aversion]: https://en.wikipedia.org/wiki/Loss_aversion
[5-minutes]: https://signalvnoise.com/posts/3124-give-it-five-minutes

### Tracked Properties & Promises

Dependency autotracking requires that tracked getters access their dependencies
synchronously. Any access that happens asynchronously will not be detected as a
dependency.

This is most commonly encountered when trying to return a `Promise` from a
tracked getter. Here's an example that would "work" but would never update if
`firstName` or `lastName` change:

```js
class Person {
  @tracked firstName;
  @tracked lastName;

  @tracked
  get fullNameAsync() {
    return this.reloadUser().then(() => {
      return `${this.firstName} ${this.lastName}`;
    });
  }

  async reloadUser() {
    const response = await fetch('https://example.com/user.json');
    const { firstName, lastName } = await response.json();
    this.firstName = firstName;
    this.lastName = lastName;
  }

  setFirstName(firstName) {
    // This should cause `fullNameAsync` to update, but doesn't, because
    // firstName was not detected as a dependency.
    this.firstName = firstName;
  }
}
```

One way you could address this is to ensure that any dependencies are consumed
synchronously:

```js
@tracked
get fullNameAsync() {
  // Consume firstName and lastName so they are detected as dependencies.
  let { firstName, lastName } = this;

  return this.reloadUser().then(() => {
    // Fetch firstName and lastName again now that they may have been updated
    let { firstName, lastName } = this;
    return `${this.firstName} ${this.lastName}`;
  });
}
```

However, **modeling async behavior as tracked properties is an incoherent
approach and should be discouraged**. Tracked properties are intended to hold
simple state, or to derive state from data that is available synchronously.

But asynchrony is a fact of life in web applications, so how should we deal with
async data fetching?

**In keeping with Data Down, Actions Up, async behavior should be modeled as
methods that set tracked properties once the behavior is complete.**

Async behavior should be explicit, not a side-effect of property access. Today's
computed properties that rely on caching to only perform async behavior when a
dependency changes are effectively reintroducing observers into the programming
model via a side channel.

A better approach is to call a method to perform the async data fetching, then
set one or more tracked properties once the data has loaded. We can refactor the
above example back to a synchronous `fullName` tracked property:

```js
class Person {
  @tracked firstName;
  @tracked lastName;

  @tracked
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }

  async reloadUser() {
    const response = await fetch('https://example.com/user.json');
    const { firstName, lastName } = await response.json();
    this.firstName = firstName;
    this.lastName = lastName;
  }
}
```

Now, `reloadUser()` must be called explicitly, rather than being run implicitly
as a side-effect of consuming `fullName`.

### Accidental Untracked Properties

One of the design principles of tracked properties is that they are only
required for state that _changes over time_. Because tracked properties imply
some overhead over an untracked property (however small), we only want to pay
that cost for properties that actually change.

However, an obvious failure mode is that some property _does_ change over time,
but the user simply forgets to annotate that property as `@tracked`. This will
cause frustrating-to-diagnose bugs where the DOM doesn't update in response to
property changes.

Fortunately, we have a strategy for mitigating some of this frustration. It
involves the way most tracked properties will be consumed: via a component
template. In development mode, we can detect when an untracked property is used
in a template and install a setter that causes an exception to be thrown if it
is ever mutated. (This is similar to today's "mandatory setter" that causes an
exception to be thrown if a watched property is set without going through
`set()`.)

Unfortunately this strategy cannot be applied to values accessed by tracked
getters. The only way we could detect such access would be with native
[Proxies](proxy), but proxies are more focussed on security over flexibility
and recent discussion shows that [they may break entirely when used with
private fields](https://github.com/tc39/proposal-class-fields/issues/106). As
such, it would not be ideal for us to

## Alternatives

### Ship tracked properties in user-land

Instead of shipping `@tracked` today, we can focus on formalizing the primitives
which it uses under the hood in Glimmer VM (References and Validators) and make
these publicly consumable. This way, users will be able to implement tracked in
an addon and experiment with it before it becomes a core part of Ember.

This approach is similar to the approach taken with component managers in the
past year, which unblocked experimentation with `SparklesComponent`s as a way to
validate the design of `GlimmerComponent`s, and unlocked the ability for power
users to create their own component APIs. However, the reference and validator
system is a much more core part of the Glimmer VM, and it could take much longer
to figure out the best and safest way to do this without exposing too much of
the internals. It would certainly prevent `@tracked` from shipping with Ember
Octane.

### Keep the current system

We could keep the current computed property based system, and refactor it
internally to use references only and not rely on chains or the old property
notification system. This would be difficult, since CPs are very intertwined
with property events as are their dependencies. It would also mean we wouldn't
get the DX benefits of cleaner syntax, and the performance benefits of opt-in
change tracking and caching.

### We could keep `set`

Tracked properties were designed around wanting to use native setters to update
state. If we remove that constraint and keep `set`, it opens up some
possibilities. There is precedent for this in other frameworks, such as React's
`setState`.

However, keeping `set` likely wouldn't be able to restrict the requirement for
`@tracked` being applied to all mutable properties for the same reason `get`
must be used in interop - there's no way for a tracked property to know that a
plain, undecorated property could update in the future.

### Allow explicit dependencies

We could allow `@tracked` to receive explicit dependencies instead of forcing
`get` usage for interop. This would be very complex, if even possible, and is
ultimately not functionality `@tracked` should have in the long run, so it would
not make sense to add it now.

### We could wait on private fields and Proxy developments

Native [Proxies](proxy) represent a lot of possibilities for automatic change
tracking. Other frameworks such as Vue and Aurelia are looking into using
recursive proxy structures to wrap objects and intercept access, which would
allow them to track changes without _any_ decoration. We also considered using
recursive proxies in earlier drafts of this proposal, even though they aren't
part of our support matrix we believed they could be used during development to
assert when users attempted to update untracked properties which had been
consumed from tracked getters.

However, as mention above, TC39 has made it clear that this was [not an intended
use for Proxy](https://github.com/tc39/proposal-class-fields/issues/106), and
they will be _breaking_ this functionality with the inclusion of private fields.
They have also expressed that [they would like to solve this
use-case](https://github.com/tc39/proposal-class-fields/issues/162#issuecomment-441101578)
(observing object state changes in general) separately, and [a strawman proposal
was made](https://github.com/littledan/proposal-proxy-transparent) (though it
has not advanced and does not seem like it will). We could wait to see what the
future looks like here, and see if we can provide a more ergonomic tracked
properties RFC in the future.

[proxy]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy
