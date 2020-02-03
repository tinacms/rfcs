---
title: Media API
submitter: ncphillips
reviewers:
  - dwalkr
pull_request: https://github.com/tinacms/rfcs/pull/3
---

Media management is a must have feature for content management systems. 

So far we have focussed on the management of data edited through basic form inputs. Little thought has been given to the management of other media e.g. images, pdfs, videos, etc. The purpose fo this document is to (1) identify the basic media  operationos, (2) consider how these operations might be performed, and (3) describe the high level architecture of implementation.

Contents:

- Media Manager UI
  - Screen Plugin
  - Dialogue
- Fields
  - How will images be uploaded?
- TinaCMS Media API
- Implementing Media Providers
  - Client
  - Server
  - Where does adaption happen?
  - Gatsby Example
- Remaining Questionos

## Media Manager UI (React Component)

A user interface is required to manage media. This interface must support the following:

- List available media
  - Display a preview of the file
  - Paginated
- Upload
- Delete
- Insert (optional)
- "No Media Provider" view for when the developer has not setup media on their site.

This UI Component will be used two ways:

- As a Screen Plugin
- As a Dialogue

### Screen Plugin

This screen plugin will be added by default to all instances of TinaCMS.

It will basically just render the Media Manager UI with the "Insert" option hidden.

### Dialogue

Should the user need to insert an image (e.g. as a field value or as a markdown tag) they must be able to access the Media Manager. For this we need a dialogue that contains the Media Manager UI. It must be possible for the developer to progammatically open this dialogue and receive the values the user selects.

More concretely, the Media Manage Dialogue must:

- Be accessible from Inline Editing, Fields, etc.
- Open media manager and pass it props (i.e. disabled files, provider)
- `onSelect(files: File[]): void` callback

A potential abstract for interacting with this dialogue might be:

```ts
cms.media.open({
  onSelect(files) {
    console.log("Inserting", files);
  }
});
```

### Inputs

This is a note for some future work. The terms used here are somewhat lose. 

By default the Media Manager will have two inputs for uploading media:

1. A normal file upload dialogue
2. A dropzone 

It will be possible to add other "media-input" plugins to the Media Manager. For example, 
one "media-input" may allow users to select an image from Unsplash. The selected image would
then be uploaded to the site's media provider before inserting the appropriate path into the form field. 

Details of this will be explored at a later date. 

## Fields

Fields that use media will indirectly access the media through the `cms.media` interface.


**Previews:** The Image Field will default to using `cms.media.previewSrc(???)` to generate the preview URL.

```js
{
  name: "frontmatter.header",
  component: "image",
}
```

Should the user desire a more specific method of generating the preview URL (e.g. using the results of the Gatsby image query) they may override the `previewSrc` function on the file component:

```js
{
  name: "frontmatter.header",
  component: "image",
  previewSrc(formValues, fieldProps) {
    let path = fieldProps.input.name.replace("rawFrontmatter", "frontmatter")
    let gastbyImageNode = get(formValues, path)
    if (!gastbyImageNode) return ""
    return gastbyImageNode.childImageSharp.fluid.src
  }
}
```

This will make it easier for people to get started using media in TinaCMS quickly, while still providing the ability to choose more specific behaviour.

### How will media be uploaded?

This is a very rough example of how images might be uploaded from a field:

```tsx
export const ImageField = wrapFieldsWithMeta<InputProps, ImageProps>(props => {
  const cms = useCMS();

  return (
    <ImageUpload
      value={props.input.value}
      previewSrc={props.field.previewSrc(props.form.getState().values, props)}
      onDrop={file => {
        const mediaFile = await cms.media.upload(file, props);

        props.input.onChange(cms.media.src(mediaFile, props));
      }}
    />
  );
});
```

**Why is generating the `src` url separate from uploading?** This is incase theres ever a situation where the `src` to be inserted is dependent on the content being edited. For example, for Gatsby the `src` is a relative path from the current file to the image file.

## TinaCMS Media API

This change would add a new `media` property to the `TinaCMS` interface:

```ts
interface TinaCMS {
  media: {
    open(props: MediaProps): void;
    provider: MediaProvider | null
    setProvider(provider: MediaProvider): void
  }
}

interface MediaProvider {
  list(???): Promise<Media> // Returns a list of media
  upload(file: File, options: FieldProps): Promise<Media> // Upload a new file
  delete(id: string): Promise<Media> // Delete an existing file
  src(???): Promise<string> // Generate a URL for a file
  previewSrc(???): Promise<string> // Generate a preview URL for a file
}

interface Media {
  // ???
}
```

Setting up the CMS to support media would be as simple as:

```js
import SomeMediaProvider from "tinacms-some-media";

const options = {
  // ...
};

cms.media.setProvider(SomeMediaProvider(options));
```

## Implementing Media Providers

There are two parts required for supporting new media providers:

- Client: A browser object for interacting with the server
- Server: An express router that talks directly to the media provider API

### Client

These browser objects that implement the `MediaProvider` interface defined above.

### Server

The server are implemented as Express Routers. They should proxy request to the media provider's API and handle any authentication. Expect any secrets to be on the request context.

Our intention is for these to be general purpose libraries that could be used outside of TinaCMS applications.

### Where does adaption happen?
The current thought is that the proxies will be dumb proxies that simply handle authentication and pass off all requests to the media provider API.

Any code needed to adapt the media provider's API to Tina's will live in the client object.

### Gatsby Example

Here's an example of how a Gatsby plugin would then tie everything together.

**gatsby-browser.js**

```js
import Cloudinary from "tinacms-cloudinary";

cms.media.add(new Cloudinary({ endpoint: "/___cloudinary" }));
```

**gatsby-node.js**

```js
import cloudinaryRoutes from "cloudinary-proxy";

export const onCreateDevServer = ({ app }, options) => {
  app.use("___cloudinary", (req, res, next) => {
    req.CLOUDINARY_KEY = options.API_KEY;
    next();
  });
  app.use("___cloudinary", cloudinaryRoutes());
};
```

The credentials would then be set in the user's gatsby-config:

**gatsby-config.js**

```js
{
  resolve: "gatsby-tinacms-cloudinary",
  options: {
    API_KEY: process.env.CLOUDINARY_KEY
  }
}
```

