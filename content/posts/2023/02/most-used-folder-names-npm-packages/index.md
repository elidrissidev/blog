---
title: "Exploring the Most Commonly Used Folder Names in Popular NPM Packages"
date: 2023-02-22T22:11:45+01:00
draft: false
description: "I was listening to Episode 575 of Syntax podcast last week when I heard Wes talk about an idea he had for the show. It was about writing a 
script that will fetch 8000 repos and analyze the most common directories and then explain the purpose of each one. I thought it would be interesting to 
try this with NPM packages and share the result, so here it is."
cover: "cover.jpg"
useRelativeCover: true
coverCaption: "Photo by [Maksym Kaharlytskyi](https://unsplash.com/de/@qwitka?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/photos/Q9y3LRuuxmg?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)"
---

I was listening to [Episode 575 of Syntax podcast](https://syntax.fm/show/575/save-us-from-config-file-hell) last week 
when I heard Wes talk about an idea he had for the show. It was about writing a script that will fetch 8000 repos and analyze 
the most common directories and then explain the purpose of each one. I thought it would be interesting to try this with NPM packages and share the result, so here it is.

## Collecting the data

### 🏗️ Building a list of the most popular NPM packages

The first step is to gather the data from a significant number of NPM packages, this seemed easy at first because I was planning to use the [NPM Registry API](https://github.com/npm/registry/blob/master/docs/REGISTRY-API.md#get-v1search) 
to get this information, but it doesn't seem to allow searching by popularity alone. So I'm going to have to use a [manually curated list of popular packages](https://gist.github.com/elidrissidev/36e17a5f1206a656848eff086a3519e4#file-packages-json) for this purpose 🤷‍♂️.

### 📁 Fetching the folder structure of each package

Now that we have the list of packages, we will need the repository URL of each one to get their directory structure. To accomplish this, I'm going to hit the NPM Registry API to retrieve the data about each package which should contain the repository URL, then I will request the GitHub REST API to get the contents of the root folder:

```js
import packages from "./packages.json" assert { type: "json" };

const NPM_REGISTRY_API_URL = "https://registry.npmjs.com";
const GITHUB_API_URL = "https://api.github.com";

const pkgs = await Promise.all(
  packages.map(name =>
    fetch(`${NPM_REGISTRY_API_URL}/${name}`)
      .then(res => res.json())
  )
);

const repos = await Promise.all(
  pkgs.map(pkg => {
    const githubUrl = new URL(pkg.repository.url);
    const repoNamespace = githubUrl.pathname.replace(/\.git$/i, "");

    return fetch(`${GITHUB_API_URL}/repos${repoNamespace}/contents`)
      .then(res => res.json());
  })
);
```

### 📊 Counting the directories

We now have a two-dimensional array representing the contents of every package, the last step is to filter out individual files and aggregate the result to count the occurence of every directory:

```js
const directories = new Map();

repos.flat()
  .filter(file => file.type === 'dir')
  .forEach(dir => {
    let count = directories.get(dir.name);
    directories.set(dir.name, count ? ++count : 1);
  });
```

I've made the full code available in a [gist](https://gist.github.com/elidrissidev/36e17a5f1206a656848eff086a3519e4).

It's time to take a look at the result!

## Analyzing the data

{{< figure src="chart.jpg" alt="A Bar Chart that shows the mapping between the folder names and their count" caption="A Bar Chart that shows the mapping between the folder names and their count" >}}

The above chart is enough to give you an idea about the most common directories, but we're interested in more than just numbers, 
so let's dive deep and explain what's in each folder:

1. `.github`: With more than 60 occurrence, it's not surprising to see this folder on the top spot given that all of the analyzed packages are 
hosted on GitHub. `.github` is a special folder that can contain a variety of config files and templates related to [GitHub Actions workflows](https://docs.github.com/en/actions/using-workflows/about-workflows), 
as well as other organizational files for a GitHub repository such as `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md` and more. [Example from `axios`](https://github.com/axios/axios/tree/v1.x/.github).

2. `tests`/`test`/`__tests__`: Writing tests is an important practice in software development, and in NPM packages you'll often see them 
stored in one of these folders. It can also be used to hold testing helpers and utility functions. [Example from `express`](https://github.com/auth0/node-jsonwebtoken/tree/master/test).

3. `docs`: Documentation is an essential part of any package, as it provides users with the information they need to understand how to use it 
and how it works. The documentation usually includes usage instructions, API documentation, and more. It can also be included directly 
in the repository's `README.md` file, but it's often split into multiple files and stored in this folder for ease of navigation and maintenance. 
Although the documentation files can be in any format, the most common one is [Markdown](https://en.wikipedia.org/wiki/Markdown). [Example from `node-fetch`](https://github.com/node-fetch/node-fetch/tree/main/docs).

4. `lib`: The `lib` folder, short for "library", is mostly used to store the actual source code of the package, but it can also be used to store 
third-party code, utilities and helpers. [Example from `passport`](https://github.com/jaredhanson/passport/tree/master/lib).

5. `examples`: Good documentation goes well with good examples, not only does it provide a practical demonstration on how to use the package, but it also 
allows developers to quickly get up to speed and start using the package in their own projects. [Example from `express`](https://github.com/expressjs/express/tree/master/examples).

6. `src`: Similar to `lib`, the `src` folder is also used to organize code, allowing for easy access to the main codebase. [Example from `yaml`](https://github.com/eemeli/yaml/tree/master/src).

7. `scripts`: Maintaining a package can be a lot of work, there's lots of repetitive tasks that need to be done often such as building the package for 
different targets, preparing a new release, etc. This is where automation scripts can help, and if a package has any, there's a good chance you'll find 
them in this folder. [Example from `history`](https://github.com/remix-run/history/tree/dev/scripts).

8. `packages`: If you see a directory with this name, you're most likely looking at a [monolithic repository](https://en.wikipedia.org/wiki/Monorepo) 
(monorepo). Monorepos contain the code for different projects/sub-components within a single repository, this offers several benefits such as simplified 
dependency management, and improved code reusability to name a few.  [Example from `react`](https://github.com/facebook/react/tree/main/packages).

9. `bin`: Sometimes it may be desired or even crucial for a package to provide a command line interface, take a testing framework 
like `jest` as an example. NPM allows packages to [publish](https://docs.npmjs.com/cli/v9/configuring-npm/package-json?v=true#bin) executable binaries 
for this purpose, and as a convention they're usually placed in this directory. [Example from `nanoid`](https://github.com/ai/nanoid/tree/main/bin).

10. `benchmarks`: This directory contains benchmark tests that help measure the performance of the package's code, these tests can be are very useful 
when experimenting with performance optimizations, and to ensure no slowdowns are introduced between releases. [Example from `graphql`](https://github.com/graphql/graphql-js/tree/main/benchmark).

11. `.husky`: Git [hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) are custom scripts that run in response to some event (e.g. before a 
commit is created), and they can choose to abort that event under certain conditions. One of their main drawbacks though is that they live inside 
the `.git` folder, which means they cannot be directly versioned like the rest of the project. This folder is used by the popular [Husky](https://github.com/typicode/husky) 
package that makes it possible to include Git hooks with your project and it takes care of installing them to their appropriate location so they 
can be detected by Git. [Example from `uuid`](https://github.com/uuidjs/uuid/tree/main/.husky).

## Conclusion

I hope this exploration helped you gain insights about the common naming practices and conventions used in the NPM ecosystem of packages, as well as highlight their importance in improving the accessibility and readability of your project. You can use this knowledge if you plan to build your own 
package, contribute to an existing one, or simply navigate your way around if you're browsing the source code.
