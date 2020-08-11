---
rfc: 0010
title: API Events
submitter: ncphillips
reviewers: dwalkr
pull_request: 11
---

In RFC 0009 we introduced `cms.events` which gives the ability to listen for events happening within the CMS.

There are some core events that are used (i.e. `cms:enabled`) but there are also some events defined by:

- UI elements (i.e. `sidebar:opened`)
- API clients (i.e. `git:commit`).

Before the introduction of `cms.events` these objects were totally decoupled from the `cms`. They
defined their own behaviour and knew nothing of the `cms` itself or of any other part. However, in order
for them to define their own events they would have to be aware of `cms.events`.

This RFC attempts to address the above issue for API clients by proposing a change to the way APIs
are registered with the CMS. The current `CMS#registerApi` method is incredibly simple:

```ts
class CMS {
  registerApi(name: string, api: any): void {
    this.api[name] = api;
  }
}
```

## Proposal

> If an `api` object has an `events` attribute that is an instance of `EventBus` it will subscribe
> to all events from that object and pass them through to all subscribers of `cms.events`

The change to the `CMS` class would essentially be as follows:

```diff
class CMS {
  registerApi(name: string, api: any): void {
+   if (api.events instanceof EventBus) {
+     api.events.subscribe(this.events.dispatch);
+   }
    this.api[name] = api;
  }
}
```

API clients that wish to broadcast events to the CMS could do the following:

```ts
import { EventBus } from '@tinacms/core';

export class MyApiClient {
  events = new EventBus();

  async saveFile(data) {
    try {
      // Make the request
      this.events.dispatch('myapi:save:success');
    } catch {
      this.events.dispatch('myapi:save:failure');
    }
  }
}
```

With this approach CMS Events continue to always go through the `cms.events` object, which allows subscribers to be ignorant of the origin of events. For example, any subscribers to the `github` events don't need to interact directly with an instance of `GithubClient`. This makes it possible for multiple instances of `GithubClient` to be registered with the CMS, and still all subscribers will be properly notified.

The APIs also benefit from being decoupled from the `CMS` object, making it easier to manage the relationship between the two.
