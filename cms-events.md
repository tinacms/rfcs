---
title: CMS Events
submitter: ncphillips
reviewers:
pull_request:
---

# CMS Events

## Background

The purpose of the core `CMS` object is to provide a centralized control point that allows various parts of the application to be decoupled. There are currently two ways that the `CMS` helps with this:

- The `cms.api` provides adapters to external services.
- The `cms.plugins` allow for the behaviour of the CMS to extended or modified.

A common problem to be solved is that of message passing between decoupled parts. For example, if a screen plugin is added to the CMS the sidebar must be notified so that it can re-render the menu.

In order to support this kind of behaviour, any object that you might be interested in must implement it's own subscription API. Furthermore, if you want to create a new thing to subscribe to then that object must be attached to the CMS.

## Problem

1. There is no standard subscription API
1. There is no standard place for subscribing to events in TinaCMS.
1. There is no standard approach of dispatching events.
1. There is no real message passing, it's just notifications about state change.

## Proposal

In order to facilitate message passing between decouple pieces of the CMS we propose adding an event bus to the core cms. Events can be dispatched to the `events` object, which then forwards them to any listeners in the CMS. The `events` object itself is entirely stateless.

### Examples

_Note: These are example uses. The exact details of the Plugin events in particular are not a part of this RFC._

**Log all events**

```ts
cms.events.subscribe((event) => {
  console.log(event);
});
```

**Log all Plugin events**

```ts
cms.events.subscribe(
  ({ plugin }) => {
    console.log(`Something happened to the plugins`);
  },
  ['plugins']
);
```

**Log when any Plugin is added or removed**

```ts
cms.events.subscribe(
  ({ plugin }) => {
    console.log(`Added Plugin of type "${plugin.__type}" called "${plugin.name}"`
  },
  [
    "plugins:add",
    "plugins:remove",
  ]
)
```

**Log when a Form Plugin is added**

```ts
cms.events.subscribe(
  ({ plugin }) => {
    console.log(`Added form "${plugin.__type}" called "${plugin.name}"`
  },
  ["plugins:add:form"]
)
```

**Log any Form Plugin event**

```ts
cms.events.subscribe(
  ({ plugin }) => {
    console.log(`Added form "${plugin.__type}" called "${plugin.name}"`
  },
  ["plugins:*:form"]
)
```

**Dispatching Events**

```ts
cms.events.dispatch({
  type: 'some-event-name',
})
```

**Dispatching an "Error" Event**
```ts
new Form({
  onSubmit(...) {
    return cms.api.github.commit(...)
      .catch(error => cms.events.dispatch({
        type: 'errors',
        error
      }))
  }
})
```

**Logging Errors External Service**

```ts
function logErrorsExternally(event: Event, cms: CMS) {
  cms.api.logger.error(event.error);
}

cms.events.subscribe(logErrorsExternally, ['errors']);
```



**React Hook**

```tsx
function SidebarList() {
  const [screens, setScreens] = useState<ScreenPlugin>([]);

  useEvent(
    (event, cms) => {
      setSreens(cms.getType('screen').all());
    },
    ['plugins:*:screens']
  );

  return (
    <ul>
      {screens.map((screen) => {
        return <li>{screen.label}</li>;
      })}
    </ul>
  );
}
```

### Core Interfaces

```ts
interface EventBus {
  subcribe(listener: Listener, subscription: Subscription): Unsubscribe;
  dispatch(event: Event): void;
}

type Listener = (event: Event, cms: CMS) => void;

type Subscription = string[]

type Unsubscribe = () => void;
```

_Note: I have ideas for increasing the type-safety but I think that would clutter this RFC._

### Creating Higher Level Interfaces

This event system is quite low level. There is an opportunity to create higher
level APIs if that seemes useful. For example, you could still do this:

```ts
mycms.forms.subscribe((event) => {
  console.log(event);
});
```

Which could be implemented as:

```ts

interface AddForm {
  type: "plugins:add:forms"
  form: Form
}

interface RemoveForm {
  type: "plugins:remove:forms"
}

type FormEvents = AddForm | RemoveForm

class MyCMS extends CMS {
  forms: {
    subscribe: (cb) => {
      return this.events.subscribe<FormEvents>(cb, ['plugins:*:forms'])
    }
  }
}
```
