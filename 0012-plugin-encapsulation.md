---
title: Plugin Encapsulation
---

The goal of this PR is to create a simpler way for developers to include plugins like `react-tinacms-github` in their projects, without sacrificing the flexibility and composability of the current plugin system.

## What is a plugin?

A `plugin` in Tina, registered to `cms.plugins`, is a small, single-responsibility chunk of code. A plugin **type** (for example, `form`) represents a single attachment point in Tina where these single-responsibility chunks of code can execute.

Contrast this to something like a WordPress plugin, in which the term describes a more holistic solution that will leverage multiple attachment points in WordPress to do a variety of things, ideally all under the umbrella of a cohesive feature (for example, an online store.)

There is a tremendous advantage to being composition-oriented. Plugins don't _have_ to be these huge monolothic things. At the same time, there is benefit to increasing the convenience and reducing the mental overhead of bootstrapping a plugin like `react-tinacms-github`. There's no reason we can't have it both ways.

## Proposed Changes

A high-level view of bootstrapping `react-tinacms-github` into your project involves the following changes to your codebase:

1. Adding serverless functions to authenticate and proxy requests to the GitHub API
2. Configuring an instance of `GithubClient` and registering it to the `TinaCMS` instance via `cms.registerApi`
3. Wrapping your application in `TinacmsGithubProvider`
4. Attaching toolbar plugins
5. Using the provided hooks to load/persist form data

Steps 2-4 could conceivably be consolidated by adding code to `TinacmsGithubProvider` that creates + registers the `GithubClient` instance, and invokes `useGithubToolbarPlugins`. This would have the downside of making `TinacmsGithubProvider` far more prescriptive, making it harder for developers going "off-menu" to use the provider for their own purposes.

### Initializer Plugins

The solution is to create a new type of plugin that can register other plugins. Developers then have the option of piecing together the `react-tinacms-github` tools themselves, or using this "meta-plugin" to do it for them.

An **initializer plugin** contains an `init` and, optionally, a `deinit` method. These methods are passed the CMS instance they were registered to, allowing them to register more plugins, register APIs, and subscribe to events.

```ts
interface InitializerPlugin extends Plugin {
  __type: 'initializer'
  init: (cms: CMS) => void
  deinit?: (cms: CMS) => void
}
```

The CMS will subscribe to `plugin:add:initializer` events to intercept the plugin and call its `init` method, and subscribe to `plugin:remove:initializer` to call the outgoing plugin's `deinit` method. This will ensure plugins registered by an initializer plugin are added and removed correctly when the corresponding initializer plugin is added or removed, preserving the intent of the [dynamic plugin system](https://tina.io/blog/dynamic-plugin-system/).

#### Example

The contrived example below consolidates the work of attaching the `GithubClient` and registering toolbar plugins:

```ts
function GithubInitializer(options): InitializerPlugin {
  return {
    __type: 'initializer',
    init: (cms) => {
      cms.registerApi('github', new GithubClient(options.clientOptions))
      cms.plugins.add([BranchSwitcherPlugin, RepoPlugin])
    },
    deinit: (cms) => {
      cms.registerApi('github', null) // currently no good way to do this ¯\_(ツ)_/¯
      cms.plugins.remove(BranchSwitcherPlugin)
      cms.plugins.remove(RepoPlugin)
    },
  }
}

// _app.js
const cms = new TinaCMS({
  //...
  plugins: [GithubInitializer(pluginOptions)],
})
```

### Wrapper Plugins

Extending Tina may involve adding components to the render tree, such as with `react-tinacms-github`'s `TinacmsGithubProvider`. For components that need to wrap the tree at no particular depth (such as with context providers,) or components that don't need to wrap the entire tree but need to place a component (such as a modal) in the tree at no particular position, **wrapper plugins** can automatically ensure these components are included in the component tree without a developer needing to specifically include them. Wrapper plugins behave similar to [wrapRootElement in Gatsby](https://www.gatsbyjs.com/docs/browser-apis/#wrapRootElement).

```ts
interface WrapperPlugin extends Plugin {
  __type: 'wrapper'
  wrap: (children: React.ReactNode) => React.Component
}
```

The order in which wrapper plugin components will appear in the stack is undefined and should not be relied upon. If your plugin depends on a specific ordering of wrappers, compose them together in a single wrapper plugin.

All wrapper plugins are guaranteed to be nested inside of TinaProvider, so they can access the CMS object with `useCMS()`.

#### Example

```tsx
function GithubWrapperPlugin({onLogin, onLogout, error}): WrapperPlugin {
  __type: 'wrapper',
  wrap: children => (
    <TinacmsGithubProvider onLogin={onLogin} onLogout={onLogout} error={error}>
      {children}
    </TinacmsGithubProvider>
  )
}

// _app.js
const cms = new TinaCMS({
  //...
  plugins: [
    GithubWrapperPlugin(providerOptions)
  ]
})
```

### Non-contrived example

To demonstrate the potential change in DX of bootstrapping `react-tinacms-github`:

#### Current

```tsx
//...
import {
  GithubClient,
  TinacmsGithubProvider,
  GithubMediaStore,
} from 'react-tinacms-github'

export default function App() {
  const cms = React.useMemo(() => {
    const github = new GithubClient({
      proxy: '/api/proxy-github',
      authCallbackRoute: '/api/create-github-access-token',
      clientId: process.env.NEXT_PUBLIC_GITHUB_CLIENT_ID,
      baseRepoFullName: process.env.NEXT_PUBLIC_REPO_FULL_NAME,
      baseBranch: process.env.NEXT_PUBLIC_BASE_BRANCH,
      authScope: 'repo',
    })
    return new TinaCMS({
      enabled: false,
      toolbar: true,
      sidebar: false,
      apis: {
        github,
      },
      media: new GithubMediaStore(github),
    })
  }, [])

  return (
    <TinaProvider cms={cms}>
      <TinacmsGithubProvider
        onLogin={() => {
          //...
        }}
        onLogout={() => {
          //...
        }}
        error={null}
      >
        <Page />
      </TinacmsGithubProvider>
    </TinaProvider>
  )
}
```

#### Proposed

```tsx
//...
import { GithubSuite } from 'react-tinacms-github'

const clientConfig = {
  proxy: '/api/proxy-github',
  authCallbackRoute: '/api/create-github-access-token',
  clientId: process.env.NEXT_PUBLIC_GITHUB_CLIENT_ID,
  baseRepoFullName: process.env.NEXT_PUBLIC_REPO_FULL_NAME,
  baseBranch: process.env.NEXT_PUBLIC_BASE_BRANCH,
  authScope: 'repo',
}

const providerConfig = {
  onLogin: () => {
    //...
  },
  onLogout: () => {
    //...
  },
  error: null,
}

export default function App() {
  const cms = React.useMemo(() => {
    const cms = new TinaCMS({
      enabled: false,
      toolbar: true,
      plugins: [
        new GithubSuite({
          clientConfig,
          providerConfig,
        }),
      ],
    })
    return cms
  }, [])

  return (
    <TinaCMSProvider cms={cms}>
      <Page status={status} error={error} />
    </TinaCMSProvider>
  )
}
```

Wrapper plugins make it practical once again to use the alternative `withTina` HOC syntax in projects that use `react-tinacms-github`:

```tsx
//...
import { GithubSuite } from 'react-tinacms-github'
import App from 'next/app'

export default withTina(App, {
  enabled: false,
  toolbar: true,
  plugins: [
    new GithubSuite({
      clientConfig: {
        proxy: '/api/proxy-github',
        authCallbackRoute: '/api/create-github-access-token',
        clientId: process.env.NEXT_PUBLIC_GITHUB_CLIENT_ID,
        baseRepoFullName: process.env.NEXT_PUBLIC_REPO_FULL_NAME,
        baseBranch: process.env.NEXT_PUBLIC_BASE_BRANCH,
        authScope: 'repo',
      },
      providerConfig: {
        onLogin: () => {
          //...
        },
        onLogout: () => {
          //...
        },
        error: null,
      },
    }),
  ],
})
```
