+++
title = 'Shortcomings of npm/yarn and reasons for recommending pnpm'
date = 2022-01-10T11:52:46+09:00
description = 'npm vs yarn vs pnpm'
+++

## What is pnpm

According to the official website of [pnpm](https://pnpm.io/), pnpm stands for performant npm.

> Fast, disk space efficient package manager

So, pnpm is similar to npm/yarn. Currently (December 2021), many major open source projects ([vue](https://github.com/vuejs/vue-next), [prisma](https://github.com/prisma/prisma)...) use pnpm. This article will take a closer look at the shortcomings of npm/yarn and how pnpm solved them.

## Conclusion

npm/yarn - Shortcomings

- The flat node_modules structure allows access to any package that is not referenced.
- Packages from different projects cannot be shared, resulting in disk space consumption.
- Installation is slow, and there are duplicates installed in node_modules.

pnpm - Solution

- Using a unique node_modules structure with symbolic links, only those in package.json can be accessed (strict).

- Packages to be installed are hard-linked from the global store, saving disk space (efficient).

The above measures also make installation faster (fast).

Strict, efficient, and fast monorepo support are also said to be features of pnpm from the official website. However, since npm8 and yarn also support monorepo, I don't think it is a shortcoming. I will talk a little about pnpm's monorepo support at the end.

## Disk space

### npm/yarn- Disk space consumption of node_modules

npm/yarn has a shortcoming in that it uses too much disk space. If you install the same package 100 times, 100 packages will be stored on the disk in node_modules. In everyday life, if the previous project is finished and node_modules is left as it is, it often uses a lot of disk space. To solve this, [npkill](https://npkill.js.org/) is often used.

```shell
$ npx npkill
```
You can scan all node_modules under the current folder and dynamically delete them.

### pnpm - Efficient disk space

On the other hand, pnpm stores packages in the same folder (content-addressable store), and when you install the same version of the same package again, it just creates a hard link. The default location on MacOs is ~/.pnpm-store. Moreover, if there are different versions of the same package, only the differences are newly saved. Then, when you install, if it is in the store, it will be reused, and if not, it will be downloaded and saved in the store.

What I was able to do by using hard links

- Installation is very fast (faster than yarn's pnp mode in benchmarks)

- Save disk space

Below is the output when reinstalling express on a computer that has previously installed it. I'll also post the output when installing npm/yarn.

pnpm

```
$ pnpm i express
Packages: +52
++++++++++++++++++++++++++++++++++++++++++++++++++
Progress: resolved 52, reused 52, downloaded 0, added 0, done

dependencies:
+ express 4.17.1
```

npm

```
$ npm i express
npm WARN npm@1.0.0 No description
npm WARN npm@1.0.0 No repository field.

+ express@4.17.1
added 50 packages from 37 contributors and audited 50 packages in 4.309s
found 0 vulnerabilities
```

yarn

```
$ yarn add express
yarn add v1.22.11
[1/4] 🔍 Resolving packages...
[2/4] 🚚 Fetching packages...
[3/4] 🔗 Linking dependencies...
[4/4] 🔨 Building fresh packages...

success Saved lockfile.
success Saved 29 new dependencies.
info Direct dependencies
└─ express@4.17.1
info All dependencies
├─ accepts@1.3.7
├─ array-flatten@1.1.1
├─ body-parser@1.19.0
├─ content-disposition@0.5.3
├─ cookie-signature@1.0.6
├─ cookie@0.4.0
├─ destroy@1.0.4
├─ ee-first@1.1.1
├─ express@4.17.1
├─ finalhandler@1.1.2
├─ forwarded@0.2.0
├─ inherits@2.0.3
├─ ipaddr.js@1.9.1
├─ media-typer@0.3.0
├─ merge-descriptors@1.0.1
├─ methods@1.1.2
├─ mime-db@1.51.0
├─ mime@1.6.0
├─ ms@2.0.0
├─ negotiator@0.6.2
├─ path-to-regexp@0.1.7
├─ proxy-addr@2.0.7
├─ raw-body@2.4.0
├─ safer-buffer@2.1.2
├─ serve-static@1.14.1
├─ type-is@1.6.18
├─ unpipe@1.0.0
├─ utils-merge@1.0.1
└─ vary@1.1.2
✨ Done in 1.14s.
```

pnpm makes it easy to see how many packages are reused and how many new downloads have been made, so I think it's a little easier to understand the output.

## Node_modules structure and dependency resolution

Now, consider the same simple example: installing a package foo that depends on bar.
npm/yarn has had three major updates to reach its current form. Let's take a look at each one to understand the improvements to pnpm.

### npm1 - nested node_modules

Since foo depends on bar, the simplest way to think about it is to put bar in foo's node_modules.
npm1 uses the same concept, so the structure looks like this.
```
.
└── node_modules
└── foo
├── index.d.ts
├── package.json
└── node_modules
└── bar
├── index.js
└── package.json
```

If bar has other requests, such as lodash, they will be included in bar's node_modules, which are called nested node_modules. So what are the problems with this structure?

```
.
└── node_modules
└── foo
├── index.js
├── package.json
└── node_modules
└── bar
├── index.js
├── package.json
└── node_modules
└── lodash
├── index.js
└── package.json
```

Yes. This tends to be infinitely nested. If the structure becomes too deep, the following problems will occur.

- The path is too long and exceeds the path length limit of Windows.
- A large number of duplicate installations will occur. If foo and bar have a dependency on the same version of loadsh, when you install it, separate node_modules will have the exact same lodash.
- The same instance value cannot be shared. For example, if you quote React from a different place, it will become a different instance, so the internal variables that should be shared cannot be shared.

### npm3/yarn - flat node_modules

npm3 (and yarn) adopted flat node_modules and has been used until now. Node.js's [dependency analysis](https://nodejs.org/api/modules.html#all-together) algorithm has a rule that if it cannot find a package in node_modules in the current directory, it will recursively analyze the parent directory's node_modules. By using this, all packages are placed in node_modules directly under the project, and problems with packages that cannot be shared and dependencies that are too long were solved.

The above example has the following structure.

```
.
└── node_modules
├── foo
│ ├── index.js
│ └── package.json
└── bar
├── index.js
└── package.json
```

This is also the reason why about 50 packages are created in node_modules if you install only express.

However, a new problem arises.

1. You can access packages that are not written in package.json ([Phantom](https://rushjs.io/pages/advanced/phantom_deps/)).

2. The uncertainty of installing node_modules ([Doppelgangers](https://rushjs.io/pages/advanced/npm_doppelgangers/) - hallucinations of seeing your own image).

3. The flat node_modules algorithm itself is complex and takes time.

#### Phantom

If you install foo, which has a dependency on bar, bar will also be installed.
