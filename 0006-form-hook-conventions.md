---
title: RFC 0006 – Creating and Registering Forms in React
submitter: ncphillips
reviewers:
pull_request: https://github.com/tinacms/rfcs/pull/7
---

# Creating & Registering Forms in React

Creating and registering forms is a key piece of TinaCMS and right now it's awkward.

## Background

There is one main hook:

- `useForm`: Creates a form and subscribes to it's changes.

When it comes to each data sources there is more variety. The pattern is essentially this:

- `use{Format}Form`: Create a Format form.
- `useLocal{Format}Form`: Create a Format form and register with CMS.
- `useGlobal{Format}Form`: Create a Format form and register a screen plugin thath makes the form editable via the Global menu.

In this document I will use `useFormatForm` to describe an abstract form helper.

## Problem

This convention makes it incredibly awkward to create new form hooks. Every you do so it is not 1 but 3 different helpers that must be created. Feedback also suggests that the word `Local` is confusing. It is often understood as "local as in file system".

## Proposal

The convention will be to create and register forms in a two step process:

1. Create the Form
2. Register the Form

With this proposal, the hooks for the "create" step will keep the same naming convention:

```
use{Format}Form
```

The names currently used for creating & registering forms will be discarded.

Forms will be registered as plugins in two ways:

```
useFormPlugin(form)
```

or

```
useFormScreenPlugin(form)
```

**Form Plugins**

```tsx
const [, form] = useFormatForm(...)

useFormPlugin(form)
```

**Form Screen Plugins**

```tsx
const [, form] = useFormatForm(...)

useFormScreenPlugin(form)
```
