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
| 1     | name               | Unique name for the plugin which can be simple string.                                                                                                                                                                                                                                                                                                   |
| 2     | plugin             | This should be an object of type [prosemirror plugin](https://prosemirror.net/docs/ref/#state.Plugin) this will be added to the prosemirror editor when editor state is built.                                                                                                                                                                           |
| 3     | menu               | This is a react component which will be added to menubar at the top of the editor. The component will receive as prop `view` which is object of type [EditorView](https://prosemirror.net/docs/ref/#view.EditorView). As any change is made to the editor a new version of this prop will be passed to the component.                                    |
| 4     | popup              | This is an array of react components, if the feature requires any popup / floating toolbar this object can be passed. The component will receive as prop `view` which is object of type [EditorView](https://prosemirror.net/docs/ref/#view.EditorView). As any change is made to the editor a new version of this prop will be passed to the component. |
| 5     | schema             | Schema is an object with properties nodes and marks, which are in-turn key -> value objects of nodes and marks.                                                                                                                                                                                                                                          |
| 6     | markdownTranslator | This will be function which will be passed the node / mark object for the schema defined and it should return the markdown string for it.                                                                                                                                                                                                                |

## Example

A example of a embed plugin which can be used to show video from youtube:

Embed plugin code, code for embedPlugin, menuComponent, embedSchema is detailed below:

```
const embedPlugin {
    name: 'embed',
    plugin: embedPlugin,
    menu: menuComponent
    schema: embedSchema
}
```

Schema for embed node:

```
const embed = {
  group: 'block',
  attrs: { src: { default: '' } },
  draggable: true,
  parseDOM: [
    {
      tag: 'div[data-tina-embed]',
      getAttrs(domNode) {
        return { src: domNode.getAttribute('data-tina-embed') };
      },
    },
  ],
  toDOM(node) { return ['div', { 'data-tina-embed': node.attrs.src }] },
};
export default { nodes: [embed] };
```

Component added to menubar:

```
import React, { useState } from 'react';
import { EditorView } from 'prosemirror-view';

const MenuComponent = ({ view: EditorView }) => {
    const [popupVisible, setPopupVisible] = useState(false)
    const [videoSrc, setVideoSrc] = useState('')

    const insertVideo = () => {
        const { state, dispatch } = view;
        const { tr, selection } = state;
        const { $from, $to } = selection;
        const { embed } = state.schema.nodes;
        dispatch(
          tr.replaceRangeWith($from.pos, $to.pos, embed.create({ src: videoSrc }))
        );
        view.focus();
        setPopupVisible(false);
    };

    return (
      <>
        <Popup visible={popupVisible}>
            <input value={url} onChange={evt => setVideoSrc(evt.target.value)} />
            <button value="Insert video" onClick={insertVideo} />
        </Popup>
        <button value="Add video" onClick={() => setPopupVisible(true)} />
      </>
    );
  }
}
```

Node view and prosmirror plugin object:

```
import { Plugin, PluginKey } from 'prosemirror-state';

class VideoView {
  constructor(node) {
    this.dom = document.createElement("div");
    this.dom.className = "nib-video-wrapper";
    this.dom.innerHTML = `<iframe src="${node.attrs.src}" width="640" height="360" frameborder="0" allowfullscreen></iframe>`;
  }
}

export const videoPluginKey = new PluginKey('video');
export default new Plugin({
  key: videoPluginKey,
  props: {
    nodeViews: {
      embed(node, view) { return new VideoView(node, view) },
    },
  },
});
```
