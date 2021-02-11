# "Tina Anywhere" Hooks

This RFC proposes the introduction of a `@tinacms/react-anywhere` package that will contain React hooks that can be used by components to understand and interact with a Tina editing environment. Hooks provided by `@tinacms/react-anywhere` will *always* return useful values, even when not in a Tina environment (in other words, outside of `TinaProvider`.)

## Motivation

It can be hard to organize code for components that need to be Tina-aware. The current way to determine if a page is in "edit mode" is to call the `useCMS` hook to get the CMS instance, and check its `enabled` property:

```
import { useCMS } from 'tinacms'

const MyComponent = () => {
  const cms = useCMS()
  return <h1>The CMS is {cms.enabled ? 'enabled' : 'disabled'}</h1>
}
```

This creates an **implicit dependency** on `MyComponent`; your code will throw an error if `MyComponent` is not nested inside of an instance of `TinaProvider`. This is, in general, good defensive programming; however, we've discovered that "knowing if the user wants to be editing things right now" is pretty fundamental to a lot of decisions that a Tina-aware UI might need to make.

### Event Bus as the Primary Thing-Doer

Passing the CMS enabled state this way is trivial and marginally useful, but the real goal here is to hydrate the event bus using the same approach. I see the event bus as a needed "air gap" to allow components to understand and interact with Tina, such that code can be organized without concern toward whether a given component will ultimately be wrapped in a `TinaProvider`. This approach, then, provides an alternate solution to the problem described in https://github.com/tinacms/rfcs/pull/15/files

## API

```ts
import { useCMSEnabled } from '@tinacms/react-anywhere'

const MyComponent = () => {
  const enabled = useCMSEnabled()
  return <h1>The CMS is {enabled ? 'enabled' : 'disabled'}</h1>
}
```

In this example, when `MyComponent` is used outside of `TinaProvider`, `enabled` will always be `false`. When used inside of a `TinaProvider` component, `enabled` will match the value of `cms.enabled`.

## Implementation

https://github.com/tinacms/tinacms/pull/1742

Values returned by "anywhere" hooks will be provided via React Context. Context allows us to specify default consumer values for when there is no corresponding provider, which can specify sensible default values that will gracefully hydrate into useful information when components *are* nested in a provider.

**@tinacms/react-anywhere/src/use-cms-enabled.ts**
```ts
import * as React from 'react'

export const CMSEnabledContext = React.createContext(false)
export const useCMSEnabled = React.useContext(CMSEnabledContext)
```

**tinacms/src/components/TinaProvider.tsx**
```tsx
// ...
import { CMSEnabledContext } from '@tinacms/react-anywhere'

export const TinaProvider({ cms, children }) => {
  return (
    <CMSContext.Provider value={cms}>
      <CMSEnabledContext.Provider value={cms.enabled}>
        {children}
      </CMSEnabledContext>
    </CMSContext>
}
```
