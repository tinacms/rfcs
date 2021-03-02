---
title: Tina Content API - use .ts files for Tina Cloud config
---

# Tina Content API - use .ts files for Tina Cloud config

Tina's Content API is based on the Forestry schema configuration which was traditionally located in the `.forestry` folder for a project. To use it in a Tina project we've essentially just renamed the folder to `.tina` and kept everything else the same. This has helped us to get started quickly but it's opened up the door for a lot of confusion and put an extra burden on us to double-document the source of truth for a Tina form. We've had a lot of discussions about moving to a `.json`-based or preferrably a `.js/.ts`-based config, where we can expose Typescript types to make configuration easier.

Using JavaScript files would allow users to write their template definitions like the OSS Tina form configs, reducing the confusion and giving developers a familiar feel to what they might be used to.

The trouble with using `.js/.ts` config is that our server would need to execute those files in order to know what the config is.While this is probably something we can do down the road, it's too large of an impact to take on right now. So I propose that we just add a cli command to compile `.ts/.js` files in the `.tina` folder to `.yml`.

A definition would look like this:

```ts
// .tina/settings.ts
import type { TinaCloudSettings } from "tina-graphql-gateway";

const settings = (): TinaCloudSettings => ({
  sections: [
    {
      path: "content/posts",
      label: "Posts",
      templates: ["post"],
    },
  ],
});

export default settings;
```

And a template:

```ts
// .tina/templates/post.ts
import type { TinaCloudTemplate } from 'tina-graphql-gateway'

const template = (): TinaCloudTemplate => ({
  label: 'Post',
  fields: [{
    label: "Title",
    name: 'title',
    type: 'text' // we need this, but it'd be the built-in Tina plugins, not Forestry's fields
    component: 'my-custom-component' // they can still provide this, if left off we'll use `type`
  }]
})

export default template
```

One of the benefits we've maintained from using traditional `.forestry` config has been that we enforce a `type` field on each field definition for a template, so a template looks like this:

```yaml
---
label: Author
fields:
  - label: Name
    name: name
    type: text # we know the expected type from this
```

In Tina, there's no concept of a `type` field, instead you have a `component` field, which offers no insight about the type of field. Strong types are essential to the Content API working properly, so we'd need to add an additional field to the traditional Tina form config. We could pass the `component` value through if it's provided, otherwise fall back to the `type`.

## Caveats

### Runtime code

We don't have a mechanism (right now) to pull in these definitions to the runtime Tina form, so things like `parse`,`format` and `validate` would be a no-op for this stage of the feature. And `component`, which can be either a string or a React component, would need to be a string.

## Risks

### Still slightly different to Tina

While this is much closer to Tina, there are still a few differences required to make this work.

### Pages reference

Right now the `.yml` definitions store a `pages` field, which lists each page the template controls. For now I think it's easiest to continue to write those to the generated `.yml` file, if someone wants to mess with it manually they can go in there to do it. This would keep it out of the user-facing configuration.
