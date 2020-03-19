# MDX Support

## Summary

| MDX is an authorable format that lets you seamlessly write JSX in your Markdown documents. You can import components, such as interactive charts or alerts, and embed them within your content. This makes writing long-form content with components a blast 🚀.

-- from mdxjs.com

Tina's position as a toolkit for persisting content from the perspective of where it is rendered puts it in a position where something like mdx feels like a natural fit.

### Features of MDX

#### **jsx**

The ability to drop in structured content to an otherwise flat `markdown` file is compelling. CMSs have been doing this for years via shortcodes. Much of this RFC can be applied to rich shortcode UX support as well.

#### Imports & exports

MDX supports not only the ability to import components into markdown files, but also to reference other markdown files from within markdown. It's incredibly powerful and could have implications for content relationships within a CMS if done properly. But this RFC doesn't address importing and exporting, if done right it's one of those things that should Just Work® - but as a first attempt at seeing how this would work I've left this out of the equation.

## Concerns

MDX is so powerful that it comes with a set of downsides

### Coupling to the environment

Markdown is environment-agnostic, MDX is very opinionated - and while it's "just javascript", it requires a lot of tooling and much of the ecosystem assumes React is at the center of things. For this reason, this proposal is aimed at addressing React-specific needs.

#### Mixing presentation logic with content

I think this is a valid concern in many cases - but the disconnect of headless CMS content and it's view is what makes Tina so compelling in the first place. We've been using shortcodes for years and this is a chance to make them better. It would be the job of this editor to provide helpful error boundaries but that's not outlined in this RFC.

### Editor experience (ie. editors don't want to write JSX)

In past discussions on MDX it seems that everyone agrees that editors don't want to write JSX, nor did they want to write Wordpress or Hugo shortcodes. They just wanted structured content that their view could do something with. This proposal makes it a point that while files may be saved with JSX in them - the editor won't actually display JSX to the user, it will instead offer a GUI which spits out JSX behind the scenes to be rendered.

## Proposal

This is a 2-part proposal

1. Provide an editor which passes the rendering phase back to the user (inversion of control), this way the user is responsible for node. The proof of concept was built using Slate.js
2. Provide a GUI for editing JSX - this is in the form of a customizable form schema that spits out JSX

### 1 - The Editor

Create an editor which can render user-provided components as editable nodes.

```js
const Renderer = props => {
  switch (props.type) {
    case "h1":
      return <h1 {...props} />;
    case "h2":
      return <h2 {...props} />;
    case "img":
      return <img {...props} />;
    case "jsx":
      return <JsxRenderer {...props} />;
    default:
      return <p {...props} />;
  }
};

// More details below
const JsxRenderer = props => {
  switch (props.type) {
    case "Img":
      return <Img {...props} />;
    case "Gallery":
      return <Gallery {...props} />;
  }
};

// This registers which components can be rendered and provides instructions for the GUI
const schemaMap = {
  Img: {
    props: {
      src: { kind: "string", required: true }
    }
  },
  Gallery: {
    props: {
      children: { kind: "array", component: "Img", min: 1, max: 5 }
    }
  }
};

// ....

<Editor
  renderElement={Renderer}
  initialValue={mdxString}
  schemaMap={schemaMap}
/>;
```

For non-editing states, pass the identical list of components to the a `MDXProvider`, instead of a `jsx` key, pass the custom components directly:

```js
export const MyPage = () => {
  const mdxMap = {
    h1: props => <MyCustomH1 {...props} />,
    // ...
    Gallery: props => <Gallery {...props} />,
    Img: props => <Img {...props} />
  };

  // ...

  <MDXProvider components={mdxMap}>{mdxString}</MDXProvider>;
};
```

A component is responsible for toggling (this can obviously be another component we provide)

```js
export const MyPage = () => {
  const [inEditMode, setInEditMode] = React.useState(false);

  return inEditMode ? (
    <Editor
      onExitEditMode={() => setInEditMode(false)}
      components={editorMap}
      initialValue={mdxString}
      {...otherEditorProps}
    />
  ) : (
    <>
      <SomeEditButton onClick={() => setInEditMode(true)}>
        Edit Me
      </SomeEditButton>
      <MDXProvider components={mdxMap}>{mdxString}</MDXProvider>
    </>
  );
};
```

Since the underlying nodes are identical components (both user-provided) this should look the same regardless of edit vs normal mode

This allows us to completely remove the editor for production builds because the rendering is not driven by the form but by the components individually.

#### Rendering JSX from within the Editor view

When the editor parses the mdx string it will come nodes with types like `h1`, `p`, and finally `jsx`:

```
<Gallery>
  <Img src="https://example.com/1.png" />
  <Img src="https://example.com/2.png" span={2} />
</Gallery>
```

It was stated above that we're passing the rendering back to the user (inversion of control) - but what about a jsx string? First we need to bring it to life by parsing it into valid javascript, then since we have access to the registered components we can just look them up and pass the props to them. We'll wrap jsx components in a "void" element - one that won't allow text to be edited inside it (more on `TinaHoverForm` below):

```js
// returns a big jsx ast
const jsxBlob = acorn.parse(jsxString); // or babel or our own lightweight parser

// Note that this is going to be more complex than what's demonstrated here
// and there's probably the need for sanitizing this
return (
  <TinaHoverForm>
    {jsxBlob.items.map(item => {
      const Component =
        componentMap[item.name] || throw "No component provided";
      return <Component {...item.attributes}>{item.children}</Component>;
    })}
  </TinaHoverForm>
);
```

Now whenever our editor comes across a JSX node it can use this function to render it.

So if this works according to plan we have an editor which visually behaves the same as an `MDXProvider`. You can toggle back and forth between edit and normal modes and since they're driven by the same components they should behave the same.

### 2- The JSX GUI

> When we introduce a gui to make inserting nontrivial components easy, it to some degree doesn't matter whether we use portable text or mdx or blocks (even tho, as Nolan mentioned, blocks exists at a higher altitude than those other two)Supporting mdx in a wysiwyg is still an interesting proposition for the same reason forestry is: developers don't have to change how they work if they don't want, and we can just slap a user friendly interface on top that exports to the preferred formatBut I also think this is the kind of thing that even developers would prefer to use a gui for, if it were good enough - DJ

It's an awful experience in any non-IDE environmnent. One thing that most agree on is if we're going to support JSX we need a GUI which aids the user in inserting the markup. This can be a form inside a popup window which spits out a JSX string when it's schema is valid.

The form is based on the `schemaMap` provided to the component,

- First you pick which component you want to insert,
- Based on that - we load the appropriate form which ensures the user has provided all values in the schema.
- Once the form is valid it takes the form data (which is a javascript object) and transforms it into a JSX string which is stored in the exact same way as other markdown nodes are.
- The rendering kicks in and the process described above renders JSX as real react components

```js
// These schemas will drive our GUI form so editors don't have to write JSX
const schemaMap = {
  Img: {
    props: {
      src: { kind: "string", required: true }
    }
  },
  Gallery: {
    props: {
      children: { kind: "array", component: "Img", min: 1, max: 5 }
    }
  }
};

export const MyPage = () => {
  const parsedMdFile = useMdx(mdFile);
  return (
    <Editor
      schemaMap={schemaMap}
      renderElement={Renderer}
      initialValues={parsedMdFile}
    />
  );
};
```

The `schemaMap` tells our `TinaHoverForm` what type of form to build and how to validate it. An invalid form should never be passed on as `jsx`. As you can see, it's capable of spitting out nested items, in this case a `Gallery` has an array of `Img` as it's children. A completed form might look like this:

```js
const formData = {
  type: "jsx",
  children: [
    {
      type: "Gallery",
      children: [
        {
          type: "Img",
          props: { src: "https://example.com/1.png", span: 2 }
        }
      ]
    },
    {
      type: "Img",
      props: { src: "https://example.com/1.png" }
    }
  ]
};
```

When we save it we transform it to a JSX string (with some sort of helper library most likely)

```js
const jsxString = `
<Gallery>
  <Img src="https://example.com/1.png" />
  <Img src="https://example.com/2.png" span={2} />
</Gallery>
`;
```

### Hover form editing (fig 2)

The idea here is that the editing experience is mostly the same as what we're used to. All other peices of markdown editing work as usual, but you can prompt the `TinaHoverForm` by typing a `<` (similar to slash commands in Slack or Notion). Typing `<` will bring up a hover form (fig 3) which will ask you which of your registered components you'd like to insert, based on your selection the rest of the form will then be built according to the `schemaMap`

A valid submission of the hover form will result in a 2-step sequence:

1. Transform the form data to a `jsx` string (fig 4)
2. Insert the string into the editor as a `jsx` node (fig 8)

![](./mdx-rfc.png)

## Further Questions

### Unified ecosystem

MDX is part of the [Unified](https://unifiedjs.com/) project - meaning it's a pluggable parser which can expand to offer many customizations, but the AST it works with differs from that of Slate's model. Slate is very flexible in this regard, it offers custom serializers so theoretically we could probably blend the editor and MDX renderer together. This would open up the doors to the editor so it could control every aspect of a markdown file, including anything a Unified plugin might control - for example a toolbar could have some sort of 'generate Table of Contents' button leverage this plugin [this plugin](https://github.com/remarkjs/remark-toc) to spit out a table of contents. There are also some amazing things it can do with [natural language processing](https://github.com/retext-project/retext), like having the ability analyze text for curse-words or grammatical mistakes.

### Tooling

This proposal is based on the use of the Slate editor, though there's more tooling and familiarity around Prosemirror internally, Slate's schema-less model makes working with `jsx` nodes more simple from a beginner's perspective. More experienced users of Prosemirror can probably get this working with similar ease. An editor we work with needs to support the following:

- Provide a mechanism for listening for certain patterns (ie. Typing `<` should indicate we may want to start a JSX block) and provide our own UI around it. For example many editors will support a "tooltip"-like pop which allow you to add link attributes to an `a` tag, we need the popup to house a complex form.
- Allow custom node types - for example every editor likely knows that a `p` tag is a paragraph node, but we need to supply a tag called `jsx`
  - Allow a section of the editor to be void - text inside a jsx tag should not be editable like the rest of the nodes, it needs to be fenced in.
- Provide some sort of `insert` function which allows us to insert arbitrary nodes (in our case the node shape is a JSX block)

### Security

Not sure, we'll need to sanitize anytime we move strings into executable code, opinions welcome
