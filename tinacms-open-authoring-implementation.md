---
title: open-authoring-implementation
submitter: jamespohalloran
reviewers:
pull_request: 
---

# Open Authoring

This RFC proposes how Open Authoring will be implemented onto a new site.


## Packages

### alerts
Display modals with prompts when initialContent fails to load, or actions fail due to the Github-state

### github-error-actions
Helper for describing the UI that gets shown in a modal based on the Github request error.

### github-auth
Provides function for requesting an auth code from Github's API, and exchanging it for an access token.

### open-authoring
Provides base OpenAuthoringProvider component, which manages the state for if we are connected to Github / if fork exists.


## Implementation:

```ts
// YourLayout.ts
import authenticate from '@tinacms/github-auth/authenticate'
import OpenAuthoringProvider from '../open-authoring/open-authoring/OpenAuthoringProvider'

const enterEditMode = () =>
  fetch(`/api/preview`).then(() => {
    window.location.href = window.location.pathname
  })

const exitEditMode = () => {
  fetch(`/api/reset-preview`).then(() => {
    window.location.reload()
  })
}

const YourLayout = ({ Component, pageProps }) => {
  return (<OpenAuthoringProvider
      authenticate={() => authenticate('/api/create-github-access-token')}
      enterEditMode={enterEditMode}
      exitEditMode={exitEditMode}>
      {...children}
    </OpenAuthoringProvider>)
}
```

```ts
// YourSiteForm.ts

const YourSiteForm = ({ form, children }) => {
  return (
    <>
      <FormAlerts form={form} /> {/* Show form success or fail messages */}
      {children}
    </>
  )
}

```

// You will also need a few Github Specific pages to handle auth:
```ts
// api/create-github-access-token.ts
import createAccessToken from '@tinacms/github-proxy-backend/create-access-token'

export default createAccessToken

```

// You will also need a few Github Specific pages to handle auth:

```ts
// api/create-github-access-token.ts
import createAccessToken from '@tinacms/github-proxy-backend/create-access-token'

export default createAccessToken
```

```ts
import useGithubAuthRedirect from '@tinacms/github-auth/useGithubAuthRedirect'

export default function Authorizing() {
  useGithubAuthRedirect()

  return (
      <h2>Authorizing with Github, Please wait...</h2>
  )
}

```


