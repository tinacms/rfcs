---
title: @tinacms/react-core
submitter: ncphillips
reviewers:
-  dwalkr
pull_request: 
---

# @tinacms/react-core

## Summary

This is an RFC to:

1. Rename `react-tinacms` to `@tinacms/react-core`
1. Create a new `react-tinacms` that wraps `tinacms` and provides helpers for the concepts it introduces.

## Motivation

The organization of code in `tinacms`, `react-tinacms`, and `@tinacms/core` has resulted in some some awkward dependency management and a confusing API. 

Note that I have I have listed these in order from highest-to-lowest level of abstraction. The order then is odd, as the name  `react-tinacms` somewhat breaks the convention where a package named `react-thing` is usually a wrapper for `thing`. The breaking of this convention is what causes the confusion. Since the `tinacms` package introduces new concepts outside of `@tinacms/core` (I.e. screens, global forms, content creators) the helpers for those concepts don’t belong in the lower level `react-tinacms`. As a result, devs who are not-intimately familiar with the structure of the TinaCMS codebase have difficulty knowing which package they should be importing from. Developers should not need to know at what layer a concept is introduced in order to guess which package they need to import from. The exception to this is the case of datasource specific helpers like `useRemarkForm` from `gatsby-tinacms-remark`.

## Proposal

To address this issue, I suggest Moving the current contents of `react-tinacms` to `@tinacms/react-core` and repurposing `react-tinacms` as a wrapper of the higher level `tinacms`, thus inverting their relationship. 

With this change: 

1. `@tinacms/react-core` will only wrap core behavior and be very generic. It will know nothing of the various concepts introduced at higher levels (i.e. screens, global forms, etc.)
2. `tinacms` will depend on only the `@tinacms/react-core` wrapper 
3. `react-tinacms` will wrap `tinacms` and redeclare helpers (e.g. `useCMS`) to be more type-aware of the concepts introduced by the `tinacms` package. It will also provide custom hooks for concepts introduced in `tinacms` (e.g. `useScreen`, `useGlobalForm`, `useContentCreator`, etc.)

Perhaps the helpers for higher level tinacms concepts will actually be defined in tinacms itself (since it’s already React) and then just re-exported for conventions sake. 

