---
title: @tinacms/wysiwym-plugins
submitter: jpuri
reviewers:
  - ncphillips
pull_request: https://github.com/tinacms/rfcs/pull/10
---

# @tinacms/react-core

## Summary

This is an RFC to:

1. Make wysiwym extensible but adding plugins.

## Motivation

Tina's Wysiwym editor is build using ReactJS and [Prosemirror](https://prosemirror.net/). It currently supports a standard set of blocks like headings, blockquotes, images and inline styles like linking strong, italic, code. The editor provides very intuitive GUI and content is converted to markdown.

This project will make Wysiwym extensible for any specific block type / style that a user may want to support. We will have a defined api for the plugins, user can create that plugin and pass it as prop to Wysiwym react component.

## Proposal

It would be made possible to pass an array of custom plugins to Wysiwyg React component as below.

```
  <Wysiwyg
    input={{
      value,
      onChange,
    }}
    format="html"
    plugins=[videoPlugin, ...]
  />
```

Each plugin will be a simple JavaScript object with following properties defined:

| S.No. | Name               | Description                                                                                                                                                                                                                                                                                                                                              |
| ----- | ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1     | plugin             | This should be an object of type [prosemirror plugin](https://prosemirror.net/docs/ref/#state.Plugin) this will be added to the prosemirror editor when editor state is built.                                                                                                                                                                           |
| 2     | Menu               | This is a react component which will be added to menubar at the top of the editor. The component will receive as prop `view` which is object of type [EditorView](https://prosemirror.net/docs/ref/#view.EditorView). As any change is made to the editor a new version of this prop will be passed to the component.                                    |
| 3     | Popup              | This is an array of react components, if the feature requires any popup / floating toolbar this object can be passed. The component will receive as prop `view` which is object of type [EditorView](https://prosemirror.net/docs/ref/#view.EditorView). As any change is made to the editor a new version of this prop will be passed to the component. |
| 4     | schema             | Schema is an object with properties nodes and marks, which are in-turn key -> value objects of nodes and marks.                                                                                                                                                                                                                                          |
| 5     | markdownTranslator | This will be function which will be passed the node / mark object for the schema defined and it should return the markdown string for it.                                                                                                                                                                                                                |
