+++
title = 'Javascript Monorepo Development Improvement Approaches'
date = 2023-01-16T09:25:36+09:00
description = 'Javascript monorepo development improvement approaches'
+++

Monorepo (monorepo) refers to a pattern where all code of an application or microservice is stored in a single monolithic repository (usually Git).

Until now, both backend and frontend have been managed in the same repository, the so-called JavaScript monorepo. The [yarn workspace](https://classic.yarnpkg.com/en/docs/yarn-workflow) function is mainly used to share the backend/frontend and logic code, and also to avoid the need to switch between the two repositories. No more need to send out code reviews to multiple repositories. We were able to develop quickly because we only needed to clone and modify one repository.

![Multi-Repo vs Monorepo](https://storage.googleapis.com/zenn-user-upload/0b6954535479-20240604.png)

However, looking back at the situation a year ago, there are two major problems.

- Lack of [yarn 1](https://classic.yarnpkg.com/lang/en/) (henceforth referred to as yarn) functionality (see [last year's article](https://xingyahao.com/posts/npm-yarn-pnpm/) for details) and new The inability to create a new project in the same repository as a workspace package.
- The developer experience is poor, as it takes more than 120 seconds to start up the local server for the environment.

Once there was a debate in the team about whether to go with mono-rep or multi-rep, but at that time we chose multi-rep in favor of the project schedule. However, we found that the efficiency of development was lower than mono-repo, as common components were difficult to share, and code review for development across multiple repositories was difficult.

I put my team in charge of these improvement tasks, and we have taken various measures to improve them this year.

- We were able to create new packages under the original monorepo, and we were able to reuse existing logic, etc. in a simple way.
- Local server startup time has been drastically reduced to 50% of the original time, improving the developer experience and increasing development speed.

# Various initiatives

## Creating a shared repository

At the time, we were still in the process of deciding whether to go mono-repo or multi-repo, and since code-sharing was the issue at hand, we used [Github private npm registery](https://docs.github.com/en/packages/working-with-a- github-packages-registry/working-with-the-npm-registry), I was able to share the code that was copied and pasted in multiple places as npm packages. Using [Lerna](https://lerna.js.org/), I created backend/frontend/common packages for each, moved the necessary logic to the appropriate package in multiple repositories, updated it, and published the new version via CI. I made it so that the new version would be published by CI.

However, when I actually put it into practice, I found the following problems.

- When you update the code in the shared repository, you need to wait for CI to update the new version.
- The version of the project repository is frequently updated
- Educating new engineers on how to use the shared repository
- Logic is distributed and difficult to debug

Recognizing that these are the characteristics of shared repositories and therefore hard to solve, we have decided to continue with the monorepository policy.

## Migrate to yarn berry

Since yarn's [hoist](https://classic.yarnpkg.com/blog/2018/02/15/nohoist/) is the main workspace problem, I tried a number of things to solve the hoist. First, I used yarn's nohoist, but the result was fundamentally unresolvable.

Then I looked for an alternative to yarn, and yarn berry and pnpm are candidates. Considering the cost of migrating from the existing yarn, I decided to go with yarn berry. Currently, [nmHoistingLimits](https://yarnpkg.com/configuration/yarnrc#nmHoistingLimits) is set as workspaces, and the requested hoist specifies up to the package root, and the existing mono You can create a new workspace in the repo. I have also benefited from yarn berry in many ways, such as faster installs and easier management of patches.

In addition, there are some things about yarn berry that I haven't implemented yet, such as [Plug'n'Play](https://yarnpkg.com/features/pnp) and [Zero-Installs](https://yarnpkg.com/features/zero-installs). I have not adopted it because it has changed too much, but I will keep an eye on the community support for it in the future. We are also planning to introduce a plugin to measure the execution time of npm scripts as an observability to improve the developer experience.

## Introducing Nx

Since I had been using only the yarn workspace functionality, I found that executing the same command on multiple packages was redundant and difficult to use, so I decided to introduce a monorepo tool to make monorepo easier to use.

We decided to introduce a monorepo tool to make monorepo easier to use. nx, Turborepo, and Lerna were in our consideration. nx was introduced because of its features such as parallel task execution, caching of calculation results, dependency visualization, and dependency environment analysis. nx is divided into nx core and nx plugins, and currently only nx core is installed. nx plugins are used in the nx core plugins. The installation is very simple. Installation is very simple, just create a new `nx.json`, add dependencies to each `package.json`, and you are almost done. The rest is to modify the existing npm script to use nx.

We have not introduced nx plugins, but since we expect more microservices in the future, we may use the generator and executer of nx plugins to create new projects from templates.

## Introducing swc

Up to now, starting up the monorepo development environment has been consolidated into a single command, which is very simple as a new engineer. However, it was very time consuming and the trial and error loop for engineers was very slow. We split up each task and found that the most time consuming part was compiling the TS files.

To solve this, instead of tsc, there was an improvement to use esbuild and swc. backend used [Nestjs](https://nestjs.com/), and nestjs needed the typescript emitDecoratorMetadata, esbuild does not support it, swc does, so I decided on swc. Just a note, swc does not do typecheck, so it is essential to use tsc in CI to do typecheck.

The result of compile time for nearly 3500 TS files was `tsc: 40 seconds` vs `swc: 1 second`, a significant improvement.

## Upgrade common tools and common configuration files

The versions and configuration files of the common requests for each package are scattered, for example, Typescript has different versions and configurations, and a different syntax that can be used. I can't apply new features using the old major version of `Pettier 1.x`. Mainly, the following requests

- Typescript
- Jest
- Eslint
- Prettier
- Nestjs

First, I upgraded to the same latest version as possible. In addition, we have created one main configuration file (`tsconfig.json`, `.eslintrc`), which the other packages will inherit. Use `yarn up -i` to keep the same version for future upgrades.

If you update Linter or Formatter or modify its settings, you may be afraid of git blame, which is hard to see in diffs, so you may not be able to change it easily. Actually, there is [git blame --ignore-revs-file](https://git-scm.com/docs/git-blame#Documentation/git-blame.txt---ignore-revs-fileltfilegt), bulk fixes can be ignored; Github and mainstream editors are supported.

## First step to microservice

We originally adopted monorepo, but the backend API side is almost a monolithic configuration, and the business logic and other code is in a very complex phase. Also, in order to prevent a single function from becoming very heavy due to increased memory and CPU usage, we came up with the idea of decoupling one service as a single package, taking advantage of the benefits of mono-repo.

As a first step, we are implementing notification functions such as email and mobile push notifications independently, with each type of notification having a common interface. The original API required implementation to organize and send the information to be sent, but now, using the notification service, the necessary data can be sent to the notification side, and the SDK for sending each type of notification will be migrated to the notification side, making it very simple. The notification service also uses the pub/sub model for decoupling.

Currently there are three microservices, but more microservices such as payment are possible in the future.

## Introducing circleci dynamic config and private orbs

Originally, circleci configuration file could not be split, and all CI for mono-repo was written in one file, exceeding 2300 lines, which was very difficult to understand, and CI for each package was not possible even though it was mono-repo. Then, [dynamic config](https://circleci.com/docs/dynamic-config/) and [private orbs](https://circleci.com/blog/building-private-orbs /).

dynamic config is a setup workflow that is added before the normal workflow and generates a separate configuration file like the original with this tamming. private orbs is a set of common logic like public orbs and provides the necessary functions as commands. The user only needs to pass this command and the necessary parameters and they are done.

We will continue to make CI for each microservice independent, and also consolidate existing legacy CI such as lint and test into private orbs commands, aiming to make it more convenient to add and modify new applications.

# Conclusion and future moves

We have made a lot of improvements throughout the year, and I think we have achieved good results, but there are still many areas that can be improved. For example, the next OKRs are to improve the development environment by getting the server up and running faster and decreasing the time to run Unit tests. Merging into the main monorepo to benefit projects that already exist in separate, split repositories. Terraform from [HCL](https://github.com/hashicorp/hcl) to [CDKTF](https://github.com/hashicorp/terraform-cdk) so that app engineers can also create server alerts. Migrated and added to monorepo, etc.
