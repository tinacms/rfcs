# Background

This RFC builds off of a previous one - [https://github.com/tinacms/rfcs/pull/17](https://github.com/tinacms/rfcs/pull/17). The goal of the previous RFC was to simplify the API for users designing their schema. It centered around the thought that we can abstract away all of the Forestry-specific concepts and just use Tina ones. So instead of a `yml` template with a field of `type: field_group` - we expose a function with an api that takes a shape closer to Tina:

```
export default defineSchema({
  sections: [
    {
      label: "Pages",
      name: "pages",
      path: "content/pages",
      templates: [
        {
          label: "Page",
          name: "page",
          fields: [
            {
              type: "group",
              label: "SEO",
              name: "seo",
              feilds: [...]
            },
            ...

```

This has been implemented and is working as expected. However I think it opens up the possibility of confusion, and I think there's more to be gained from enhancing this API

# Problem: `type` vs `component` is a mixture of concerns

There are a couple of necessary differences between the shape of a `template` in this function and the shape of [form config](https://tina.io/docs/plugins/forms/#form-configuration) in Tina.

With Tina, `component` is just an identifier to tell the CMS which React component to render for the field, it's also customizable. So even if we _knew_ the `text` component would always return a string, users want to be able to provide their own. As a result, we're using the `type` field to inform the shape of the data.

While it makes sense to use the keyword `type` here, it's a little bit confusing. Since all of the possible `types` are the core Tina field plugins (`text`, `group`, `blocks`, `tags` etc), we're using frontend concerns for backend configuration, a field like `tags` doesn't mean anything on the backend, all the backend cares about is that this is an array of strings.

## Other problems

### Serialized values only

Even if we did support `component` it would need to be the string defintion only. We don't have the ability to pass functions through this, that's because this data comes from the GraphQL API, so things like `parse` and `format` are also not supported.

### Customizing fields is still awkward

Since we build the forms for you, there's no place to say `component: "my-component"` on a given field. We may have a solution for that, but we can tackle this problem from here as well.

# Proposal

## Step 1 - Pass `component` through from `defineSchema`

`component` would be ignored by the backend entirely, so it's up to the user to ensure their field plugin is registered on the frontend

```
export default defineSchema({
  sections: [
    {
      label: "Pages",
      name: "pages",
      path: "content/pages",
      templates: [
        {
          label: "Page",
          name: "page",
          fields: [
            {
              type: "text",
              label: "Color",
              name: "color",
              component: "my-color-picker"
            },
            ...

```

This way you can still work as flexibly with Tina as you have in the past, for your custom component you just need to register the field plugin:

```graphql
cms.fields.add({
  name: 'my-color-picker',
  Component: ColorPicker,
})
```

## Step 2 - Come up with a system of primitive types

Add `component` to the `defineSchema` API, simplify `type` to a small subset of type primitives

As some have mentioned, the `type` of the field should be relatively low-level. A good example is the `color` field, while it's important for the `color` field to be presented to the user with a color picker, it's _shape_ is a string (ex. "#333"), the backend doesn't need to care about the presentation. So with this proposal the definition would look like this:

```
export default defineSchema({
  sections: [
    {
      label: "Pages",
      name: "pages",
      path: "content/pages",
      templates: [
        {
          label: "Page",
          name: "page",
          fields: [
            {
              type: "string",
              label: "Color",
              name: "color",
              component: "color-picker"
            },
            ...

```

### Primitive Types

- `string`
- `text`
- `image`
- `group`
- `blocks`
- `reference`

So for a `title` field:

```
...
{
  type: "string",
  label: "Title",
  name: "title",
  component: "text"
}
...

```

And a group field:

```
...
{
  type: "group",
  label: "Title",
  name: "title",
  component: "group",
  fields: [{
    type: "string",
    label: "Title",
    name: "title",
    component: "text"
  }]
}
...

```

Things get a little bit tricky with a list, we _could_ add a list type:

```
...
{
  type: "list",
  // we need to inform the shape of the items in the array
  itemType: "string",
  label: "Title",
  name: "tags",
  // should it be:
  component: "list",
  itemComponent: "text",
  // or
  component: "tags",
}
...

```

But this is awkard because we know that `tags` returns an array of strings. I'm not sure about the best way to design that part. Please comment!

# Risks

## Step 1 Risks

It might be more confusing, though I'd argue that the way it is now is the most confusing. Also, there won't be any type-safety for the component field, not a huge deal in my opinion.

## Step 2 Risks

This effort is in the hopes that we can make our content API less opinionated, it may be premature for this type of abstraction, but we're bumping up against conceptual models as we think about how to talk about backend concerns vs frontend ones.
