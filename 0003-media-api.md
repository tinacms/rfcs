---
title: Media API
submitter: ncphillips
reviewers:
  - dwalkr
pull_request:
---

## Questions

- How many media providers should a user be able to have configured
  with the CMS at the same time?
  - Gut says 99% of the time one is enough
  - Multiple is possible, but greatly complicates the code
    base and the user experience
- How will git/fs media manager handle co-located media
- `previewSrc(???): Promise<???>` api
- `src(???): Promise<???>` API

## Media Manager UI (React Component)

- List
  - Display a preview of the file
  - Paginated
- Upload
- Delete
- Support media providers

## Media Manager Screen Plugin

- Renders the UI

## Media Manager Dialogue

- Accessible from Inline Editing, Fields, etc.
- Open media manager and pass it props (i.e. disabled files, provider)
- `onSelect(files: File[]): void` callback

```ts
cms.media.open({
  onSelect(files) {
    console.log("Inserting", files);
  }
});
```

## Image Fields

- Select one media provider per image field
  - Field Def
    ```ts
    {
      name: "image",
      component: "media",
      provider: "cloudinary", // Default provider?
    }
    ```
- Media Provider has `preview` method, but it can be overridden
  - `gitMedia.preview(form, fieldName)`
  - `{ name: "image", component: "image", provider: "cloudinary", previewSrc: gatsbyPreview }`

## TinaCMS Media API

```ts
interface TinaCMS {
  media: MediaApi;
}

interface MediaAPI extends Plugins<Media> {
  open(props: MediaProps): void;
}
```

```ts
import GitMediaProvider from "tinacms-media-git";

cms.media.add(new GitMediaProvider());
```

## Media Provider API

```ts
interface MediaProvider {
  list(...): Promise<any> // Returns a list of media
  upload(file: File): Promise<any> // Upload a new file
  delete(id: string): Promise<any> // Delete an existing file
  preview(...): Promise<string> // Generate a preview URL
}
```

## Package Structure

```js
import Cloudinary from "tinacms-cloudinary-client";

cms.media.add(new Cloudinary());
```

### gatsby-tinacms-cloudinary

**gatsby-browser.js**

```js
import Cloudinary from "tinacms-cloudinary-client";

cms.media.add(new Cloudinary({ host: "localhost:8000/___cloudinary" }));
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

My **gatsby-config.js**

```js
{
  resolve: "gatsby-tinacms-cloudinary",
  options: {
    API_KEY: process.env.CLOUDINARY_KEY
  }
}
```
