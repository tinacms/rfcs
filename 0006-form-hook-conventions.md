---
title: forms
submitter: ncphillips
reviewers:
pull_request: https://github.com/tinacms/rfcs/pull/5
---

# React Form Hook Conventions

## Part 1: A Convention for Creating & Registering Forms with React

Creating form is a key piece of TinaCMS and right now it's awkward.

### Background

There is one main hook:

- `useForm`: Creates a form and subscribes to it's changes.

When it comes to each data sources there is more variety. The pattern is essentially this:

- `use{Format}Form`: Create a Format form.
- `useLocal{Format}Form`: Create a Format form and register with CMS.
- `useGlobal{Format}Form`: Create a Format form and register a screen plugin thath makes the form editable via the Global menu.

In this document I will use `useFormatForm` to describe an abstract form helper.

#### Problem

This convention makes it incredibly awkward to create new form hooks. Every you do so it is not 1 but 3 different helpers that must be created. Feedback also suggests that the word `Local` is confusing. It is often understood as "local as in file system".

#### Proposal

The convention will be to create and register forms in a two step process:

1. Create the Form
2. Register the Form

With thsi proposal, the hooks for the "create" step will keep the same naming convention:

```
use{Format}Form
```

The names currently used for creating & registering forms will be discarded. Instead they would use the following convention:

```
useForm{Somehow}
```

**Local Forms**

```tsx
const [, form] = useFormatForm(...)

useFormLocally(form)
```

**Global Forms**

```tsx
const [, form] = useFormatForm(...)

useFormGlobally(form)
```

## Part 2: A Convention for Creating Form Hooks

### Background

The current convention for form hooks is:

```
use{Format}Form
```

### Problem

The root problem is that in the vast majority of cases the format of the content is of little concern. What really matters is the storage mechanism.

This problem manifests itself as confusion when people think `useJsonForm` works for JSON coming from any data source, when it is really specific to the GitClient.

### Proposal

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

#### How Do We Deal With Different File Formats?

Until now we have tried to come up with a solution to this problem by means of `Composition`. This proposal instead uses the principles of `Inversion of Control` and `Sane Defaults` to solve this problem. In this explanation I will describe a generic `useGitForm` hook. The API for `useGithubForm` would be essentially identical.

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

#### Why not introduce the Format concept to the Form?

I'm not yet convinced this is a geneic problem.

It seems to me that this is only a problem for people using the file system as a storage medium. This includes Git, GitHub, GitLab, etc. For now, I happy to implement this change only at the levels of the `git` and `github` packages. If in the future we realize that this pattern is useful at a more generic level, then it can be introduced to `@tinacms/forms`.

#### Example: import { useGitForm } from "next-tinacms-git"

Below is a rough example of how a generic `useGitForm` hook would be defined.

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
}

function useGitForm<T = any>(file: File<T>, options: Options = {}) {
  const cms = useCMS();

  const id = options.id || file.relativePath;
  const label = options.label || file.relativePath;
  const fields = options.fields || [];
  const actions = options.actions || [];
  const [values, form] = useForm(
    {
      id,
      label,
      fields,
      actions,
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
    },
    { values: file.data, label }
  );

  const writeToDisk = useCallback(
    ({ values }) => {
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
