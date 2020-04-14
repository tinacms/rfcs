---
title: Creating and Registering Forms in React
submitter: ncphillips
reviewers:
pull_request:
---

# Higher Order Form Hooks

## Background

The current convention for higher order form hooks (HOFH) is:

```
use{Format}Form
```

## Problem

The problem is that in the vast majority of cases the format of the content is of little concern. What really matters is the storage mechanism.

This problem manifests itself as confusion when people think `useJsonForm` works for JSON coming from any data source, when it is really specific to the GitClient.

## Proposal

The convention for form hooks should switch to:

```
use{Storage}Form
```

This means that we would see forms with names like:

```ts
useGitForm(...)
useGithubForm(...)
useHasuraForm(...)
```

### What goes inside HOFH?

The purpose of a HOFH is to make the developer's life easier. It allows developers to quickly setup new forms with as little boiler plate as possible. HOFHs do this by providing two things:

- Reasonable Defaults
- New Abstractions

#### Reasonable Defaults

All forms for editing content from a particular storage API are going to have a set of options that will almost always be configured the same way. The `id`, `label`, and available `actions` are the main examples, but this also applies to `onSave`, `loadInitialValues`, and `reset`. They may even provide additional behaviour, such as the `writeToDisk` behaviour added to `useGitForm`.

It is important to emphasize that these options should be _defaults_. The developer ought to be able to override or disable them if they see fit.

#### New Abstractions

It is important that HOFHs provide these helpful abstractions. without severely limiting the developer's options.

**Example: Multiple File Formats for Git**

Until now we have tried to come up with a solution to this problem by means of `Composition`. This proposal instead uses the principles of `Inversion of Control` and `Reasonable Defaults` to solve this problem. In this explanation I will describe a generic `useGitForm` hook. The API for `useGithubForm` would be essentially identical.

The default behaviour would be to work with json:

```ts
const [, form] = useGitForm(jsonFile);
```

This would be equivalent to:

```ts
import { useGitForm } from 'react-tinacms-git';

const [, form] = useGitForm(jsonFile, {
  format: 'json',
});
```

For markdown you would change the format:

```ts
import { useGitForm } from 'react-tinacms-git';

const [, form] = useGitForm(jsonFile, {
  format: 'markdown',
});
```

The strings above are simply aliases to preconfigured Formats. You may instead pass one in explicitly:

```ts
import { useGitForm } from 'react-tinacms-git';
import { load, dump } from 'js-yaml';

interface Format<T = any> {
  serialize(data: T): string;
  parse(content: string): T;
}

const yaml: Format = {
  parse: (content) => load(content),
  serialize: (data) => dump(data),
};

const [, form] = useGitForm(yamlFile, {
  format: yaml,
});
```

**Why not introduce the Format concept to the Form?**

I'm not yet convinced this is a geneic problem.

It seems to me that this is only a problem for people using the file system as a storage medium. This includes Git, GitHub, GitLab, etc. For now, I happy to implement this change only at the levels of the `git` and `github` packages. If in the future we realize that this pattern is useful at a more generic level, then it can be introduced to `@tinacms/forms`.

### Example: import { useGitForm } from "next-tinacms-git"

Below is a rough example of how a generic `useGitForm` hook would be defined.

This example includes:

- New Abstraction: `Format`
- Reasonable Defaults:
  - Default config for `id`, `label`, `actions`, `onSave`, `reset`, `loadInitialValues`, etc.
  - Sets up `writeToDisk` by default and makes it possible to disable

```tsx
import { FormOptions } from '@tinacms/forms';
import { parseJson, serializeJson } from '@tinacms/json';
import { parseMarkdown, serializeMarkdown } from '@tinacms/markdown';

interface File<T> {
  sha?: string;
  relativePath: string;
  data: T; // The file contents pre-parsed into JSON.
}

interface Options<T> extends FormOptions<T> {
  format: 'markdown' | 'json' | Format<T>;
  writeToDisk?: boolean;
}

function useGitForm<T = any>(file: File<T>, options: Options = {}) {
  const cms = useCMS();
  const { writeToDisk = true } = options;

  const [values, form] = useForm(
    {
      id: file.relativePath,
      label: file.relativePath,
      fields: [],
      actions: [],
      loadInitialValues() {
        return cms.api.git
          .show(file.felativePath) // Load the contents of this file at HEAD
          .then((git: { content: string }) => {
            const parse = getParse(options.format);

            return parse(content);
          });
      },
      onSubmit() {
        return cms.api.git.commit({
          files: [file.relativePath],
          message: `Commit from Tina: Update ${file.relativePath}`,
        });
      },
      reset() {
        return cms.api.git.reset({ files: [id] });
      },
      ...options,
    },
    { values: file.data, label }
  );

  const writeToDisk = useCallback(
    ({ values }) => {
      if (!writeToDisk) return;
      const serialize = getSerialize(options.format);

      cms.api.git.writeToDisk({
        fileRelativePath: file.relativePath,
        content: serialize(values),
      });
    },
    [file.fileRelativePath]
  );

  useWatchFormValues(form, writeToDisk);

  return [values || jsonFile.data, form];
}

function getFormat<T>(format: Options<T>['format']) {
  if (typeof format === 'function') return format;
  if (format === 'markdown') return serializeMarkdown;
  return serializeJson;
}

function getParse<T>(parse: Options<T>['parse']) {
  if (typeof parse === 'function') return parse;
  if (parse === 'markdown') return parseMarkdown;
  return parseJson;
}
```
