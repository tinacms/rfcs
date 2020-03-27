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
Provides base OpenAuthoringProvider component, which manages the state for if we are connected to Github / if fork exists. When we attempt to enter edit-mode, it first verifies that we are authenticated and have a valid fork, and if not we display a modal prompting to perform auth/fork actions.


## Implementation:

Add the root OpenAuthoringProvider component to our main layout. In this case, we will use Github Auth.
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

Add alerts to our forms which prompt Github-specific action when errors occur (e.g a fork no longer exists).
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

You will also need a few Github Specific pages to handle auth...

Github auth callback page. 
```ts
//pages/github/authorizing.tsx

// Our Github app redirects back to this page with auth code

import useGithubAuthRedirect from '@tinacms/github-auth/useGithubAuthRedirect'

export default function Authorizing() {
  useGithubAuthRedirect()

  return (
      <h2>Authorizing with Github, Please wait...</h2>
  )
}

```

Server function (which sets the http-only access token cookie)
```ts
// api/create-github-access-token.ts

// Server function, which exchanges code and sets github auth cookie

import createAccessToken from '@tinacms/github-proxy-backend/create-access-token'

export default createAccessToken
```

And add a way to enter edit-mode from within your site.
This will first verify that we are authenticated and have a valid fork. If so, it will take us into edit-mode.

```ts
//...EditLink.tsx

import { useOpenAuthoring } from '../../open-authoring/open-authoring/OpenAuthoringProvider'

export const EditLink = ({ editMode }: EditLinkProps) => {
  const openAuthoring = useOpenAuthoring()
  return (
    <EditToggleButton
      onClick={
        editMode ? openAuthoring.exitEditMode : openAuthoring.enterEditMode
      }
    >
      {editMode ? 'Exit Edit Mode' : 'Edit This Site'}
    </EditToggleButton>
  )
}
```


