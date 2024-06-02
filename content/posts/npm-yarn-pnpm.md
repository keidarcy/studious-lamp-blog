+++
title = 'Shortcomings of npm/yarn and reasons for recommending pnpm'
date = 2022-01-10T11:52:46+09:00
description = 'npm vs yarn vs pnpm'
+++

## What is pnpm?

[pnpm](https://pnpm.io/) According to the official website, pnpm stands for performant npm.

> Fast, disk space efficient package manager

So, pnpm is similar to npm/yarn. Currently (December 2021), many major open source projects ([vue](https://github.com/vuejs/vue-next), [prisma](https://github.com/prisma/prisma)...) use pnpm. This article will look at the shortcomings of npm/yarn and how pnpm solved them in detail.

## Conclusion

npm/yarn - Shortcomings

- The flat node_modules structure allows access to any package that is not referenced.
- Packages from different projects cannot be shared, resulting in disk space consumption.
- Installation speed is slow, and there are duplicate installations in node_modules.

pnpm - Solution

- Uses a unique node_modules structure with symbolic links, so only what is in package.json can be accessed (strict).

- Packages to be installed are hard-linked from the global store, saving disk space (efficient).

The above measures also make installation faster (fast).

Strict, efficient, and fast monorepo support are also said to be features of pnpm from the official website. However, since npm8 and yarn also support monorepo, I don't think it's a shortcoming. I'll talk a little about pnpm's monorepo support at the end.

## Disk space

### npm/yarn - Disk space consumption node_modules

npm/yarn has a shortcoming in that it uses too much disk space. If you install the same package 100 times, 100 packages will be stored on the disk in node_modules. In everyday examples, if the previous project is finished and node_modules is left as it is, it often uses a lot of disk space. To solve this, [npkill](https://npkill.js.org/) is often used.

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
    │   ├── index.js
    │   └── package.json
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

If you install foo, which has a dependency on bar, you can access it directly because bar is also under node_modules.

If foo is used in a project carelessly, or if foo stops using bar one day or if you upgrade the version of bar, the state of bar referenced in the project code may change, which may cause unexpected errors.

#### Doppelgangers

Doppelgangers is a bit complicated, so in the above example, foo depends on lodash@1.0.0 and bar depends on lodash@1.0.1
```
foo - lodash@1.0.0
bar - lodash@1.0.1
```
Then, according to the nodejs [dependency analysis](https://nodejs.org/api/modules.html#all-together) rule, the PACKAGE_NAME in require(PACKAGE_NAME) must be the same as the folder under node_modules, which means that PACKAGE_NAME＠VERSION is not possible. Then the structure is

```
.
└── node_modules
    ├── foo
    │   ├── index.js
    │   └── package.json
    ├── bar
    │   ├── index.js
    │   ├── package.json
    │   └── node_modules
    │       └── lodash
    │           ├── index.js
    │           └── package.json(@1.0.1)
    └── lodash
        ├── index.js
        └── package.json(@1.0.0)
```
and
```
.
└── node_modules
    ├── foo
    │   ├── index.js
    │   ├── package.json
    │   └── node_modules
    │       └── lodash
    │           ├── index.js
    │           └── package.json(@1.0.0)
    ├── bar
    │   ├── index.js
    │   └── package.json
    └── lodash
        ├── index.js
        └── package.json(@1.0.1)
```

Which one will it be?

Both are possible...

It depends on the position in package.json. If foo is on top, you get the structure above, otherwise the structure below. This uncertainty is called Doppelgangers.

### npm5.x/yarn - Flat node_modules and lock file

To solve the uncertainty of node_modules installation, lock files were introduced. This makes it possible to have a similar structure no matter how many times you install it. This is another reason to always put lock files in version control and not edit them manually.

However, the complexity of the flat algorithm, phantom access, and performance and safety issues remain unsolved.

### pnpm - node_modules structure based on symbolic links

This part is complicated, and the explanation on the official website is the best, but I will explain it based on this.

There are two main steps before node_modules is generated.

#### Folder structure of hard links

```
.
└── node_modules
    └── .pnpm
        ├── foo@1.0.0
        │   └── node_modules
        │       └── foo -> <store>/foo
        └── bar@1.0.0
            └── node_modules
                └── bar -> <store>/bar
```

At first glance, it looks completely different from other structures, but the first node_modules only has a folder called .pnpm. Under .pnpm, a <package name@version> folder is created, and the <package name> folder under that is a hard link to the store. This alone won't work, so the next step is also important.

#### Symbolic link for request analysis

- Symbolic link to reference bar in foo
- Symbolic link to reference foo from the project

```
.
└── node_modules
    ├── foo -> ./.pnpm/foo@1.0.0/node_modules/foo
    └── .pnpm
        ├── foo@1.0.0
        │   └── node_modules
        │       ├── foo -> <store>/foo
        │       └── bar -> ../../bar@1.0.0/node_modules/bar
        └── bar@1.0.0
            └── node_modules
                └── bar -> <store>/bar
```

This is the simplest structure of pnpm node_modules. You can only quote the code in package.json, and there is no need to install anything unnecessary. [peers dependencies](https://pnpm.io/how-peers-are-resolved) is a little complicated, but everything except peers can have this kind of structure.

For example, if foo and bar depend on lodash at the same time, the structure will be as follows.

```
.
└── node_modules
    ├── foo -> ./.pnpm/foo@1.0.0/node_modules/foo
    └── .pnpm
        ├── foo@1.0.0
        │   └── node_modules
        │       ├── foo -> <store>/foo
        │       ├── bar -> ../../bar@1.0.0/node_modules/bar
        │       └── lodash -> ../../lodash@1.0.0/node_modules/lodash
        ├── bar@1.0.0
        │   └── node_modules
        │       ├── bar -> <store>/bar
        │       └── lodash -> ../../lodash@1.0.0/node_modules/lodash
        └── lodash@1.0.0
            └── node_modules
                └── lodash -> <store>/lodash
```

Now, any complex dependency can be completed with a path of this depth, making this an innovative node_modules structure.

### Solutions other than pnpm

#### npm global-style
npm also solves the problems of flat node_modules by setting [global-style](https://docs.npmjs.com/cli/v8/using-npm/config#global-style), but this solution has not spread due to the problems of the nested node_modules era.

#### dependency-check
Since it is difficult to solve the problem with npm/yarn itself, we will check it using a tool called [dependency-check](https://github.com/dependency-check-team/dependency-check).

```
$ dependency-check ./package.json --verbose
Success! All dependencies used in the code are listed in package.json
Success! All dependencies in package.json are used in the code
```
If you look at part of the official README, you will probably understand what is being done.

Compared to other solutions, pnpm is the most straightforward!

## Finally

### Basic commands
The above explanation may give you the impression that pnpm is very complicated, but in fact it is not at all!
If you have used npm/yarn before, you can use pnpm with almost no learning cost. Let's look at a few example commands.

```shell
pnpm install express
pnpm update express
pnpm remove express
```
It's almost the same as the commands you already know!

### Monorepo support

pnpm also supports monorepos. The author also has a comparison with Lerna. It would be too long to explain in detail, so I will only show one example here.

```shell
pnpm --parallel run --recursive --filter apps test
```
What it does is a command that runs npm script test asynchronously in the workspace under apps. Even in situations where you would need a monorepo management library like Lerna, you can complete it with just pnpm.