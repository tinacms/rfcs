---
title: Separating the CMS Instance Context from UI and UI Contex
---

Currently in a React app, we setup Tina by using:

- `withTina`
- `TinaProvider`

In both cases, these utilities:

- Wrap the app with a configured instance of TinaCMS stored in context
- Wrap the app with the TinaCMS UI, UI context, and business context

This is problematic for a few reasons:

- It's not possible to wrap an app with an instance of Tina without downloading all of the Tina React UI, UI context, and business logic.
- It's not possible to use Tina logic in app without downloading all of the React UI, UI context, and business logic.

## Propose API

I don't propose we change the existing API. It is the "simplest" use case.

I propose we add a few new APIs to support advanced use cases...

### The TinaCMSProvider

The TinaCMSProvider will take responsibility for holding the instance of the CMS in context, and is what the `useCMS` hook will fetch context under the hood. This can wrap the entire React app while having almost 0 impact on bundle size:


```
export function MyApp() {
 const cms = useMemo(() => new TinaCMS());
 
 return (
   <TinaCMSProvider cms={cms}>
     {...}
   </TinaCMSProvider>
 )
}
```

### The TinaUIProvider

The `TinaUIProvider` will take responsibility for rendering the Tina React UI, and UI context.

This will allow a user to wrap a sub-section of an application devoted to editing with editing logic, while leaving the `TinaCMSProvider` at the top of the application tree. This approach will greatly reduce bundle sizes by only importing and downloading Tina UI on routes that need it.


### Changes to TinaProvider

The API of `TinaProvider` wont change, but under the hood, it will now setup the `TinaCMSProvider` and `TinaUIProvider` for the user, leading to 0 breaking changes.

## Next.js Example

```
export function MyApp({Component, pageProps}) {
 const cms = useMemo(() => new TinaCMS());
 
 return (
   <TinaCMSProvider cms={cms}>
     {...}
   </TinaCMSProvider>
 )
}
```

```
export function Admin({editMode}) {
  return (
    <TinaUIProvider editing={editMode}>
      {...logic to render app}
    </TinaUIProvider>
  )
}
```

```
export function EditButton() {
  const cms = useCMS();
  
  return (
    <button type="button" onClick={() => cms.enable()}>
      {cms.enabled ? "Login" : "Logout"
    </button>
  )
}
```

## Conclusion

For general use cases, this will require no change in our docs. But for advanced use cases, such as an enterprise trying to separate all write logic from their production endpoints, this will make this trivial to do.
