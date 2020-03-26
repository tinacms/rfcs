---
title: toolbar:widget
submitter: ncphillips
reviewers:
pull_request: https://github.com/tinacms/rfcs/pull/5
---

# toolbar:widget

This RFC introduces a new Plugin type: `toolbar:widget`.

## Background

TinaCMS has several types of plugins:

- `form`: listed rendered in the Sidebar
- `screen`: listed in the global menu of the Sidebar
- `content-creator`: listed in flyout of a "New" button in the Sidebar

The tinacms.org Toolbar only reuses `content-creator`. The other UI widgets
implemented as new plugins:

- `toolbar:status`
- `toolbar:git`
- `toolbar:form-actions`

## Proposal

Introduce a new plugin type:

- `toolbar:widget`

The Toolbar will then reuse these plugin types:

- `form`
- `screen`
- `content-creator`

### `toolbar:widget` replaces `toolbar:status` and `toolbar:git`

These plugins will be replaced in favour of `toolbar:widget`.

```ts
import { Plugin } from '@tinacms/core';

interface ToolbarWidget extends Plugin {
  __type: 'toolbar:widget';
  weight: number;
  Component(props: ToolbarWidgetProps): React.Element;
}
```

If someone wanted to customize the weights of imported plugins:

```ts
import { PullRequestWidget, BranchWidget } from 'some-tina-plugin';

// Set weights as you see fit.
PullRequestWidget.weight = 1;
BranchWidget.weight = 2;

// Then register them
cms.plugins.add(PullRequestWidget);
cms.plugins.add(BranchWidget);
```

### `form` replaces `toolbar:form-actions`

This plugin type will be replaced by `form`. For now we will assume that each
page has only one `form` plugin active at a time. As is the case with the
Sidebar, the "Save" and "Reset" buttons will be assumed. The form status (dirty
or pristine) will also be assumed. Additional form actions will be provided
using the same API as that used in the Sidebar.

### `screen` comes later

Screens are React views rendered in a modal. There are two types of
modals available to Screens: `popups` and `fullscreen` modals.

In the Sidebar they're rendered in a secondary "Global" menu.

In the Toolbar a similar solution will be implemented.

### `content-creator`

This plugin type will continue to work as is.
