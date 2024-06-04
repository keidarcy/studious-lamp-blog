+++
title = 'JavaScript Monorepo 開発改善への取り組みについて(Japanese)'
date = 2022-12-16T09:25:36+09:00
description = 'Javascript monorepo development improvement approaches in Japanese'
+++


Monorepo(モノレポ)とは、アプリケーションやマイクロサービスの全コードを単一のモノリシックなリポジトリ (普通は Git) に保存するパターンを指します。

今まで backend/frontend ともに JavaScript で同じリポジトリで管理されて、いわゆる JavaScript の モノレポです。主に [yarn workspace](https://classic.yarnpkg.com/en/docs/yarn-workflow) 機能を使って、backend/frontend とロジックのコードをシェアーして、また、それぞれのレポジトリの切り替えが必要なく、コードレビューを複数に出す必要もなくなりました。一つのリポジトリさえクローンして修正すればいいので、素早く開発できました。

![Multi-Repo vs Monorepo](https://storage.googleapis.com/zenn-user-upload/0b6954535479-20240604.png)

しかし、一年前状況を振り返ったら、二つ大きな問題点があります。

- [yarn 1](https://classic.yarnpkg.com/lang/en/) (以降 yarn と呼びます)の機能不足(詳細は[去年の記事](https://xingyahao.com/posts/npm-yarn-pnpm-ja/))で新規プロジェクトを同じリポジトリに workspace の package として作れないこと。
- 環境のローカルサーバが立ち上がるのが 120 秒以上かかるなど開発者体験が悪いことです。

チーム内に一度今後の構成について、モノレポかマルチレポか議論が上がりましたが、その時はプロジェクトのスケジュールを優先してマルチレポを選びました。しかし共通コンポーネントが共有しづらいとか、複数のレポジトリにまたがる開発のコードレビューが難しいとか、開発の効率がモノレポより下がっていることがわかっていました。

これら改善のタスクは私のチームに任させて、今年に色々施策して、改善してきました。

- 新規の package を元のモノレポ配下に作ることができて、既存のロジックなどを簡易に再利用できました。
- ローカルサーバの立ち上がる時間は元の50%まで激減して、開発者体験向上や開発スピードが上がりました。

# 様々な取り組み

## shared リポジトリの作成

当時モノレポかマルチレポかまだ決まっていない段階で、取り敢えずコードシェアーが目の前の問題なので、[Github private npm registery](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-npm-registry) を利用して、複数箇所にコピペされていたコードをnpm packageにして、共有できました。[Lerna](https://lerna.js.org/) を使って、backend/frontend/共通 それぞれの package を作って複数のリポジトリに必要なロジックを該当の package 移して、更新したら、CIで新バージョンを publish と作りました。

しかし、実際に運用してみたら、以下の問題点があります。

- shared リポジトリのコードを更新したら、新しいバージョンの更新のCIを待つ必要がある
- プロジェクトのリポジトリのバージョンは頻繁に更新
- 新しいエンジニアに shared リポジトリの使い方の教育
- ロジックが分散し、デバッグが難しい

これらが shared リポジトリの特徴なので、解決しづらいと認識して、今後もモノレポの方針を固めました。

## yarn berry に移行

yarn の [hoist](https://classic.yarnpkg.com/blog/2018/02/15/nohoist/) が workspace メインの問題点なので、hoist を解決するため、色々試してみました。まず、yarn の nohoist を利用しましたが、結果根本的に解決できません。

その後、yarn の代替を探して、yarn berry と pnpm が候補です。既存の yarn からマイグレーションのコストを考えたら、yarn berry にしました。現在、[nmHoistingLimits](https://yarnpkg.com/configuration/yarnrc#nmHoistingLimits) が workspaces と設定して、依頼の hoist は package の root まで指定して、既存のモノレポに新規の workspace を作ることができます。また、インストール時が早くなったとか、patches が管理しやすくなったとか、色々 yarn berry の恩恵を受けられています。

更に、yarn berry に関して、[Plug'n'Play](https://yarnpkg.com/features/pnp) や [Zero-Installs](https://yarnpkg.com/features/zero-installs) など変わりすぎて導入していないですが、今後コミュニティのサポート状況も注目しています。また、開発者体験向上の observability として、npm script の実行時間の計測プラグインを導入する予定です。

## Nx の導入

yarn の workspace 機能だけ利用していましたので、複数の package に同じコマンドの実行は記述が冗長になったり、使いにくいところが沢山があって、モノレポをより使いやすいようにモノレポツールの導入を決めました。

Nx や Turborepo や Lerna などのツールは視野に入れました。nx はタスクの並列実行や計算結果のキャッシュ、依存関係の可視化、依存環境の解析などの機能を重視して、nx 導入しました。nx は nx coreと nx plugins が別れて、現在は nx core のみ導入しています。導入は非常にシンプルで、`nx.json`新規に作ると、各 `package.json`に依存関係を追加すれば、ほぼ終わりです。残りは既存の npm script から、nx 利用するように修正です。

nx pluginsは導入していないですが、今後は、microservice が増える想定なので、nx plugins の generator、executer を利用して、テンプレートから新規プロジェクトの作成に利用することも考えられます。

## swc の導入

今まで、モノレポ開発環境の立ち上がるのが一つのコマンドに集約しましたが、新しいエンジニアとしてとてもシンプルです。しかし、非常に時間がかかって、エンジニアの試行錯誤のループはとても遅かったです。各タスクを分割して、一番時間がかかるのが TS ファイルのコンパイルとわかりました。

解決するため、tsc の代わりに、esbuild や swc を使う改善策がありました。backend は [Nestjs](https://nestjs.com/) を利用して、nestjs は typescript の emitDecoratorMetadata が必要で、esbuild がサポートしていない、swc がサポートしているので、swc に決めました。ただ、注意点としては swc は typecheck を行わないので、CI で tsc を使って、typecheck するのが必須です。

3500近くTSファイルのコンパイル時間の結果、`tsc: 40秒` vs `swc: 1秒`となって、大幅に改善できました。

## 共通ツールのアップグレードと設定ファイルの共通化

各 package 共通の依頼のバージョンや設定ファイルが散らばって、例えば、Typescript のバージョンと設定が違って、使える文法も違いました。 `Pettier 1.x` 古い major バージョンを使って、新しい機能適用できません。主に、以下の依頼です。

- Typescript
- Jest
- Eslint
- Prettier
- Nestjs

まずは、できるだけ同じ最新のバージョンにアップグレードしました。更に、設定ファイルを大元の一個(`tsconfig.json`, `.eslintrc`)作って、他の package はそれを継承します。今後のアップグレードも `yarn up -i` を使用して、同じバージョンを維持します。

Linter や Formatter の更新や設定修正を行ったら、diff が大変 git blame が見えづないのが恐れで、なかなか変えられない方が多いかと思います。実は、[git blame --ignore-revs-file](https://git-scm.com/docs/git-blame#Documentation/git-blame.txt---ignore-revs-fileltfilegt) があって、bulk 修正が無視できます。Github と主流のエディターはサポートされています。

## microservice 化の第一歩

元々モノレポを採用していましたが、backend API 側はほぼモノリスの構成で、ビジネスロジックなどのコードは非常に複雑になっている局面です。また、一つの機能がメモリーや CPU の使用率が上がって、非常に重くなることを防ぐため、モノレポのメリットを利用して、一つサービスを一つの package として decouple する案が上がって進んできました。

第一歩として、まずはメール、モバイル Push 通知 など notification 系の機能を独立して、各種類の notification を共通の interface を持つように実装しています。元々の API 側は、送信情報の整理と送信まで実装が必要でしたが、notification サービスを利用して、必要なデータをnotification 側に送れば完了なので、各種類送信 SDK も notification 側に移行、非常にシンプルになります。また、notification サービス内にも decouple のため、 pub/sub モデルを利用しています。

現在は三つの microservice がありますが、今後も決済などの microservice が考えられます。

## circleci dynamic config と private orbs の導入

元々 circleci の設定ファイルが分割できず、モノレポの CI が全て一つのファイルに記述されて、2300行を超えて、非常にわかりづらかったし、モノレポなのに package ごとの CI はできなかったです。そこで、登場したのが [dynamic config](https://circleci.com/docs/dynamic-config/) と [private orbs](https://circleci.com/blog/building-private-orbs/)。

dynamic config は通常の workflow の前に、setup workflow 追加されて、分割した設定ファイルをこのタンミングで元のようなファイルを生成します。private orbs は public orbs のように共通のロジックをまとめて、必要な機能をコマンドとして提供して、利用する側はこちらのコマンドと必要なパラメータを渡れば終わりです。

今後は引き続き各 microservice の CI を独立して、既存のレガシーの CI の lint や test なども private orbs のコマンドに集約、新規追加や修正などをより便利にできるように目指しています。

# まとめと今後の動き

一年間通して、沢山の改善を行って、良い結果になったと思いますが、改善できるところまだまだ沢山あります。例えば、次の OKR として、開発環境の改善として、サーバをより早く立ち上がり、Unit テストの実行時間の減少があります。すでに、分割した別のリポジトリに存在しているプロジェクトを恩恵を受けるため、メインのモノレポにマージ。アプリエンジニアもサーバアラートを作成できるように、terraform を [HCL](https://github.com/hashicorp/hcl) から [CDKTF](https://github.com/hashicorp/terraform-cdk) に移行してモノレポに追加などがあります。

