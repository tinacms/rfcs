---
title: Smaller Inline API Footprint
---

The inline API is cumbersome and inserts a lot of DOM nodes that don't have any relevance to non-edit mode renders. There's a desire to clean up the work the user might have so they're production code is clean.

#### What this RFC is not

There's been discussion around 100% elimination of Tina code, this RFC doesn't address that, as most of that work needs to be done in user-land as far as I know.

## Proposed API

[See the video](https://www.loom.com/share/030d372f1b774910a3b73101cbed2486)

### Spread props

Hooks that provide a spread function which you provide **all** props to, including children. This function sets a ref on the DOM node so we can provide focus, hover controls around it, however those controls are actually mounted at the root of the dom via portal, they're just positioned over the elements with absolute positioning - this is easy because we have the ref so we know the node's coordinates.

```tsx
// The example from the video
export const Card = (props: {
  image?: string;
  hashtags?: string[];
  title?: string;
  excerpt?: string;
  reading_time?: string;
  author?: {
    name?: string;
    image?: string;
  };
}) => {
  const selectSpread = useInlineSelect({ name: "hashtag" });
  const titleSpread = useInlineTextField({ name: "title" });

  return (
    <div className="flex flex-col rounded-lg shadow-lg overflow-hidden">
      ...
      <div className="flex-1 bg-white p-6 flex flex-col justify-between">
        <div className="flex-1">
          <p className="text-sm font-medium text-indigo-600">
            <a
              {...selectSpread({
                href: "#",
                className: "hover:underline",
                children: props.hashtags.join(", "),
              })}
            ></a>
          </p>
          <a href="#" className="block mt-2">
            <p
              {...titleSpread({
                className: "text-xl font-semibold text-gray-900",
                children: props.title,
              })}
            />
            <p className="mt-3 text-base text-gray-500">{props.excerpt}</p>
          </a>
        </div>
        ...
      </div>
    </div>
  );
};
```

The API might look like this (video is an uglier version of this)

```ts
const useInlineTextField = ({ name }) => {
  const ref = React.useRef<HTMLElement>(null);
  const [active, setActive] = React.useState(false);
  // Something we could get from react-final-form?
  const { onChange } = useField(name);
  const cms = useCMS();

  React.useEffect(() => {
    if (ref.current) {
      ref.current.contentEditable = "true";
      ref.current.addEventListener("input", function (e) {
        onChange(e.target.innerText);
      });
    }
  }, []);

  const spreadProps = (props) => {
    {
      /* Early return when not in edit mode */
    }
    if (!cms.editMode) {
      return props;
    }

    return {
      ...props,
      ref,
      onClick: () => setActive(!active),
      children: (
        <>
          {/* Children is just a text node - but we're controlling it with contenteditable in this phawse */}
          {props.children}
          {ref.current && (
            <ControlTextPortal
              active={active}
              onChange={onChange}
              bounds={ref.current.getBoundingClientRect()}
            />
          )}
        </>
      ),
    };
  };

  return spreadProps;
};
```

When not in edit mode, we have an early return, so the props are only what came from the user:

```tsx
<p
  {...titleSpread({
    className: "text-xl font-semibold text-gray-900",
    children: props.title,
  })}
/>
```

```tsx
<p className="text-xl font-semibold text-gray-900" children={props.title} />
```
