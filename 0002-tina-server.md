---
title: @tinacms/react-core
submitter: 
- dwalkr
- ncphillips
reviewers:
pull_request: https://github.com/tinacms/rfcs/pull/2
---

# Tina Server

## Summary

1. Create a `tina-server` package that runs a node HTTP server in a development environment that uses Tina
2. Plugins such as `@tinacms/api-git` can hook in to this server to add API endpoints or manipulate request data
3. Users can configure `tina-server`'s behavior using a `tinacms.server.js` file that exports an `onCreateServer` function, similar to Gatsby's `onCreateDevServer`

## Motivation

* **Tina Teams Integration**

  We want to be able to pass user identity information into Tina's git backend from Tina Teams. 
  However, we don't want to bake this proprietary integration into the open-source git backend.

* **Additional Backends**

  Integrations such as Tina Teams should also be available to other backends. 
  To do this without duplicating code necessitates a highler-level abstraction to host multiple backend plugins.

This is also an opportunity to better establish rules and conventions for creating a backend integration for Tina.

## Proposal

Introduce a `tina-server` command-line package.

The `tina-server` command will start an express server. When run without arguments, the `tina-server` command will search for `tinacms.server.js` in the current working directory.
Users may give `tina-server` an optional `--config` command to specify the location of this file.

`tinacms.server.js` must export an `onCreateServer({ app })` function that will be executed after the express server is created. This API is deliberately similar to Gatsby's `onCreateDevServer` function
to make it easy for developers to configure backend behavior for both Gatsby and non-Gatsby sites.

The purpose of `onCreateServer` is to allow the developer to attach middlewares to the express app. The most common use cases will be to attach subroutes to perform API actions
and to add metadata to the request.

Middlewares that work with `onCreateServer` should be equally compatible with Gatsby's `onCreateDevServer`.

### Standalone

Note: running `tina-server` is equivalent to running `tina-server --config tinacms.server.js`

**server-config.js**

```js
/**
 * We would have some kind of standard `idenity` spec:
 */
import { identityMiddleware } from "tinacms-teams-identity"
// import { identityMiddleware } from "tinacms-serve-identity"

/**
 * These middleware provide routes for interacting with Git and Contenful respectively.
 * They are both look for the `user` object added to the request context by the which ever `identityMiddleware` is used:
 */
import { router as gitRouter } from "@tinamc/api-git"
import { router as contentfulRoutes } from "@tinamc/api-conteful"

/**
 * This middleware is defined in my repository.
 */
import { someMiddleware } from "./tinacms-server/some-middleware"

exports.onCreateServer = ({app}) => {
  app.use(someMiddleware())
  app.use(identityMiddleware())
  app.use("___tina/git", gitRouter())
}

```

### Gatsby Plugins

* gatsby-tinacms-teams-identity
* gatsby-tinacms-my-middleware
* gatsby-tinacms-git


Each of these would have their own `gatsby-node.js` file:

```js
export { identityMiddleware } from "tinacms-teams-identity"

exports.onCreateDevServer = ({ app }) => {
  app.use(identityMiddleware())
}
```

## Further Questions

* How can we ensure that different identity middlewares can interoperate with different API providers? We will need to establish a standard interface for user identification.
