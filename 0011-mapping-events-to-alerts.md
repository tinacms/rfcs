---
title: Mapping Events to Alerts
---

The `cms.alerts` is a high level abstraction built on top of `cms.events`. While `cms.events` is a generic system for communicating about events happening in the CMS, the `cms.alerts` are specifically meant for communicating with the end users.

The current use of `cms.alerts` is problematic for several reasons:

- Calls to `cms.alerts` are littered throughout the codebase and inconsistent.
- It's not possible to translate alerts without overriding the function that contains them.
- It is difficult to override or recreate functions that contain alerts (i.e. the `onSubmit` of a Form) because the developer must lookup the source code and copy the alert messages
- The same action performed from multiple places might have different alerts. (i.e. committing to GitHub)

## Proposed API

### Alert Mapping in TinaCMS

Allow users to pass in an object which maps event types to functions which return an alert:

```ts
const alerts = {
  'something-cool': () => ({ message: 'Hey Friend' }),
  'something-bad': () => ({
    level: 'error',
    message: `Good bye`,
  }),
};

const cms = new TinaCMS({
  alerts,
});
```

With this API, anytime the `something-cool` or the `something-bad` events are dispatched the corresponding function will be executed and
the resulting Alert shown to the user.

### Default Alert Maps in APIs

Any API registered with the CMS could define default alerts for its events.

```ts
class Freezer {
  events = new EventBus();

  alerts = {
    'freezer:found': () => ({ message: `We found some ${event.item}` }),
    'freezer:missing': (event) => ({
      level: 'error',
      message: `Sorry we're out of ${event.item}`,
    }),
  };

  getItem(name: string) {
    return fetchSomeItem(name).then((item) => {
      const type = item ? 'freezer:found' : 'freezer:missing';

      this.events.dispatch({ type, item: name });

      return item;
    });
  }
}

const cms = new TinaCMS({
  apis: {
    freezer: new Freezer(),
  },
});
```

### Overriding Default AlertMaps

If the default alerts are not suitable then the user can replace or remove them. In the example below the `freezer:found` event is removed entirely and the `freezer:missing` event is translated to French.

```typescript
const cms = new TinaCMS({
  apis: {
    freezer: new Freezer(),
  },
  alerts: {
    'freezer:found': null,
    'freezer:missing': (event) => ({
      level: 'error',
      message: `Désolé, nous n'avons plus de ${event.item}`,
    }),
  },
});
```

### Updating AlertMaps Dynamically

```typescript
cms.alerts.setMap({
  'freezer:found': {
    level: 'success',
    message: 'Dig in!',
  },
});
```

## Implementation Pseudocode

This would require a change to the effect of:

```typescript
interface EventsToAlerts {
  [key: string]: ToAlert;
}

type ToAlert = (event: CMSEvent) => Alert;

interface Alert {
  message: string;
  level?: AlertLevel;
  timeout?: number;
}

class TinaCMS extends CMS {
  constructor(options: TinaCMSOptions) {
    // ...

    this.alerts = new Alerts(this.events, options.alerts);
  }

  registerApi(name: string, api: any) {
    // ...

    if (this.api.alerts) {
      this.alerts.setMap(this.api.alerts);
    }
  }
}

class Alerts {
  constructor(private events: EventBus, private eventsToAlerts = {}) {
    this.events.subscribe('*', (event) => {
      const toAlert = this.eventsToAlerts[event.type];

      if (toAlert) {
        const { level, message, timeout } = toAlert(event);

        this.alerts.add(level, message, timeout);
      }
    });
  }
  setMap(eventsToAlerts) {
    this.eventsToAlerts = {
      ...this.eventsToAlerts,
      ...eventsToAlerts,
    };
  }
}
```

## Conclusion

With the above API the Alerts become decoupled from the API, allowing them to better adhere to the Open/Closed principle.

- Alerts can be replaced, translated, or removed without modifying existing packages.
- These Alerts can be replaced at runtime
- Default Alerts can be provided by the APIs that define the events.
- Existing React Componeonts and Hooks become simpler, making it easier to extend, override, or duplicate their behaviour.
- By centralizing the dispatching of Alerts, we can ensure consistency in their presentation
