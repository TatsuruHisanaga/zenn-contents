---
title: "Next.js 13.4 全文和訳 と 補足説明" # 記事のタイトル
emoji: "🐿️" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["Nextjs", "React"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # trueを指定する
published_at: 2023-05-17 08:00 # 未来の日時を指定する
---


*Written with ChatGPT-4* 

:::message alert
この記事は[Next.js 13.4 の公式リリース本文](https://nextjs.org/blog/next-13-4) をDeepLおよびChatGPT-4で翻訳・校正したのち、筆者が細かな日本語の修正を加え、体裁を整えたものです。公式リリースにある内容と齟齬がないよう、本文の翻訳には細心の注意を払っております。

 **ただし、筆者自身の学習の一環として、ChatGPT-4が生成した補足説明を【補足説明】（黄色の背景部分）として、独自の判断で追記しています。** 

最善を尽くして内容の正確性を保つよう努めていますが、日本語と英語の間の微妙なニュアンスの違いにより、いくつかの誤解を招く可能性があります。また、補足説明や和訳そのものが必ずしも正確であることを断言できません。これらの点をご理解いただけますようお願い申し上げます。
:::

---
https://nextjs.org/blog/next-13-4

# **Next.js 13.4**

2023年5月5日（金）

Next.js 13.4はApp Routerの安定性を示した、基礎的なリリースです。

- App Router (安定版)：
    - React Server Components
    - ネスト可能なルートとレイアウト
    - データ取得の簡素化
    - Streaming & Suspense
    - Built-in SEO サポート
- **Turbopack (ベータ版)**：ローカル開発サーバをより高速に、より安定的に。
- **Server Actions (アルファ版)**：クライアントJavaScriptを使用せずに、サーバー上でデータを変更することができます。

Next.js 13のリリースから6ヶ月間、我々は破壊的変更を最小限に抑えつつ、段階的に採用可能な形でNext.jsの未来の基盤を構築することに焦点を当ててきました。

本日、13.4がリリースされ、本番環境でのApp Routerの採用を開始することができるようになりました。
```
npm i next@latest react@latest react-dom@latest eslint-config-next@latest
```

## Next.js App Router
我々は2016年にNext.jsをリリースし、Reactアプリケーションをサーバーレンダリングする簡単な方法を提供し、よりダイナミックでパーソナライズされたグローバルウェブの構築を目指してきました。

[最初の発表投稿](https://vercel.com/blog/next)では、Next.jsの設計原則をいくつか共有しました：

- セットアップ不要：ファイルシステムをAPIとして利用
- JavaScriptのみ：全てが関数
- 自動的なサーバーレンダリングとコード分割
- データの取得は開発者次第

Next.jsは現在6年目を迎えています。当初の設計原則はそのままに、Next.jsがより多くの開発者や企業に採用されるにつれて、これらの原則をより良く実現するためのフレームワークの基盤的なアップグレードに取り組んできました。

我々はNext.jsの次世代版に取り組んでおり、今日の`13.4`のリリースにより、この次世代版は安定して採用可能となりました。この記事では、App Routerの設計決定と選択について詳しく説明します。


### セットアップ不要：ファイルシステムをAPIとして利用

[ファイルシステムベースのルーティング](https://nextjs.org/docs/app/building-your-application/routing)はNext.jsのコア機能でした。初期の投稿では、1つのReactコンポーネントからルートを作成するこの例を示しました：

```jsx:従来型のページルーター

//pages/about.js
import React from 'react';
export default () => <h1>私たちについて</h1>;

```

開発者は設定を追加する必要はありませんでした。ファイルを`pages/`ディレクトリに追加すれば、Next.jsのルーターが残りの部分を処理します。私たちは依然としてこのルーティングのシンプルさを愛しています。しかし、フレームワークの使用が増えるにつれて、開発者がそれを使って構築したいインターフェースの種類も増えてきました。

:::message
#### 【補足説明】
上記のコードでは、Next.jsの「ファイルシステムベースのルーティング」の一例を示しています。これはNext.jsの核となる機能で、`pages/`ディレクトリにJSXファイル（ここでは`about.js`）を追加するだけで、自動的にそのファイルがウェブルートとして扱われるというものです。具体的には、この例では`about.js`ファイルが`/about`ルートにマッピングされます。

このファイル内では、Reactコンポーネントがエクスポートされており、これがウェブページのコンテンツとなります。この例では、`<h1>私たちについて</h1>`というHTML要素が含まれているので、`/about`ルートにアクセスすると「私たちについて」という見出しのページが表示されます。

:::

<!-- 🐵第一部ここまで -->


開発者たちは、レイアウトの定義、UIレイアウトのネスティング、ローディングとエラー状態の定義について、より柔軟なサポートを求めていました。これらの改善を既存のNext.jsルーターに後付けするのは容易なことではありませんでした。

フレームワークの各部分はルーターを中心に設計されるべきです。ページ遷移、データフェッチング、キャッシュ、データの変更と再検証、ストリーミング、コンテンツのスタイリングなどがそれに当たります。

ルーターをストリーミングに対応させ、レイアウトの強化されたサポートを提供するために、私たちは新しいバージョンのルーターの構築を決定しました。

これは私たちが[レイアウトのRFC](https://nextjs.org/blog/layouts-rfc)の初版リリース後に到達した結論です。

```jsx:新 App Router ✨

// app/layout.js
export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}

// app/page.js
export default function Page() {
  return <h1>Hello, Next.js!</h1>;
}

```

ここで重要なのは、見ることができるものよりも、見ることができないものです。この新しいルーター（`app/`ディレクトリを通じて段階的に採用可能です）は、全く新しいアーキテクチャを持っており、[React Server Components](https://nextjs.org/docs/getting-started/react-essentials)と[Suspense](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming)の基盤上に構築されています。
:::message
#### 【補足説明】
上記のコードでは、Next.jsの新しいバージョンのルーターの概念を示しています。`app/layout.js`では、新しい`RootLayout`コンポーネントが定義されています。このコンポーネントは、全てのページに共通するレイアウト（ここでは`<html>`と`<body>`タグ）を定義します。`children`プロパティを使用して、具体的なページコンテンツ（ここでは`Page`コンポーネントからの出力）をこのレイアウトに注入します。

また、React Server Componentsは、クライアントサイドではなくサーバーサイドでレンダリングされるReactコンポーネントのことで、Suspenseは非同期の操作（例えばデータフェッチ）が終わるまで特定のUI（例えばローディングスピナー）を表示するためのReactの機能です。これらの技術を活用することで、より効率的なデータフェッチやローディングの管理、スムーズなページ遷移などが実現可能になります。
:::

この新しい基盤により、我々は初めてReactのプリミティブを拡張するために開発されたNext.js特有のAPIを削除できました。例えば、全体的な共有レイアウトをカスタマイズするためにカスタム`_app`ファイルを使用する必要はもうありません：

```jsx:従来型のページルーター

// pages/_app.js
// この"global layout"は全てのルートをラップします。
// 他のレイアウトコンポーネントと組み合わせる方法はありませんし、
// このファイルからグローバルなデータをフェッチすることはできません。
export default function MyApp({ Component, pageProps }) {
  return <Component {...pageProps} />;
}
```

:::message
#### 【補足説明】
このコードでは、既存のNext.jsルーターの使い方を示しています。`pages/_app.js`に定義された`MyApp`コンポーネントは、全てのページで共有されるレイアウトを定義します。しかし、ここでのレイアウトは他のレイアウトコンポーネントと組み合わせることができないため、全てのページで完全に同じレイアウトが適用されます。

また、この`MyApp`コンポーネントからグローバルなデータをフェッチすることはできません。これは、ページごとに異なるデータを必要とする場合や、特定のページでのみ必要となるデータがある場合には不便です。
:::
<!-- 第二部ここまで -->


この従来型のページルーターでは、レイアウトの組み合わせやコンポーネントと一緒にデータフェッチングを配置することができませんでしたが、新しいApp Routerではこれが可能になっています。

```jsx:新 App Router ✨
// app/layout.js
//
// ルートレイアウトはアプリケーション全体で共有されます
export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}

// app/dashboard/layout.js
//
// レイアウトはネストや組み合わせが可能です
export default function DashboardLayout({ children }) {
  return (
    <section>
      <h1>Dashboard</h1>
      {children}
    </section>
  );
}
```
　
従来型のページルーターでは、`_document`を使用してサーバーからの初期ペイロードをカスタマイズしていました。

```jsx:従来型のページルーター

// pages/_document.js
// このファイルは、サーバーリクエストに対して<html>タグと<body>タグをカスタマイズすることを可能にします
// ただし、HTML要素を直接記述するのではなく、フレームワーク特有の機能が追加されます。
import { Html, Head, Main, NextScript } from 'next/document';

export default function Document() {
  return (
    <Html>
      <Head />
      <body>
        <Main />
        <NextScript />
      </body>
    </Html>
  );
}
```

しかし、App Routerでは、Next.jsから`<Html>`、`<Head>`、`<Body>`をインポートする必要はもうありません。代わりに、直接Reactを使用します。

```jsx:新 App Router ✨
// app/layout.js
// ルートレイアウトはアプリケーション全体で共有されます
export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

<!-- 第3部ここまで -->

新しいファイルシステムルーターの構築は、ルーティングシステムに関連する他の多くの機能要求に対応する絶好の機会でもありました。以下に例を示します。

- 以前は、`_app.js`にグローバルスタイルシートをインポートする際に、`_external npm packages`（コンポーネントライブラリなど）からのみ可能でした。これは、開発者体験としては理想的ではありませんでした。App Routerでは、任意のCSSファイルを任意のコンポーネントにインポート（および配置）することができます。
- 以前は、Next.jsでサーバーサイドレンダリングを選択する（`getServerSideProps`を通じて）と、全ページがハイドレートされるまでアプリケーションとの対話がブロックされてしまいました。しかし、App Routerでは、React Suspenseと深く統合されたアーキテクチャへとリファクタリングしました。これにより、ページの一部を選択的にハイドレートし、UIの他のコンポーネントがインタラクティブであることをブロックすることなく、コンテンツをサーバーから即座にストリームできます。これにより、ページの読み込み性能を向上させることができます。

[ルーター](https://nextjs.org/docs/app/building-your-application/routing)は、Next.jsの機能を実現するコア部分です。しかし、ルーターそのものの重要性よりも、ルーターがフレームワークの残りの部分、たとえば[データフェッチ](https://nextjs.org/docs/app/building-your-application/data-fetching)といった要素をどのように統合しているかが重要です。

### JavaScriptだけ。全てが関数

Next.jsとReactの開発者は、JavaScriptとTypeScriptのコードを書き、アプリケーションのコンポーネントをまとめて使用したいと考えています。下記のコードは私たちの最初の投稿からの引用です。

```jsx
import React from 'react';
import Head from 'next/head';

export default () => (
  <div>
    <Head>
      <meta name="viewport" content="width=device-width, initial-scale=1" />
    </Head>
    <h1>Hi. I'm mobile-ready!</h1>
  </div>
);

```
<!-- ###### In future versions of Next.js, we added a DX improvement to automatically import React for you.* -->

このコンポーネントは、アプリケーション内のどこでも再利用し、組み合わせることができるロジックをカプセル化します。ファイルシステムルーティングと組み合わせると、JavaScriptとHTMLを書く感覚でReactアプリケーションを作り始める簡単な方法が得られました。
<!-- 第4部ここまで -->

例えば、データをフェッチしたい場合、Next.jsの最初のバージョンは次のようになります：

```jsx
import React from 'react';
import 'isomorphic-fetch';

export default class extends React.Component {
  static async getInitialProps() {
    const res = await fetch('https://api.company.com/user/123');
    const data = await res.json();
    return { username: data.profile.username };
  }
}
```

<!-- *~Next.jsの将来のバージョンでは、isomorphic-fetchやnode-fetchをインポートする必要がなく、クライアントとサーバーの両方でWeb fetch APIを使用できるように、fetchをポリフィルするDX改善を追加しました。~* -->

しかし、フレームワークが成熟し、採用が拡大するにつれて、私たちは新しいデータフェッチングのパターンを探求しました。

`getInitialProps`はサーバーとクライアントの両方で実行されました。このAPIはReactコンポーネントを拡張し、結果をコンポーネントのpropsに転送する`Promise`を作成することを可能にしました。

`getInitialProps`は今でも動作しますが、その後、顧客のフィードバックに基づいて次世代のデータフェッチングAPI、`getServerSideProps`と`getStaticProps`へと進化しました。

```jsx
// ルートの静的バージョンを生成する
export async function getStaticProps(context) {
  return { props: {} };
}
// または、ルートを動的にサーバーサイドレンダリングする
export async function getServerSideProps(context) {
  return { props: {} };
}
```
<!-- 第５部ここまで -->

これらのAPIによって、コードがクライアント側かサーバー側のどちらで実行されているかがより明確になり、Next.jsアプリケーションを[自動的に静的に最適化](https://nextjs.org/docs/pages/building-your-application/rendering/automatic-static-optimization)することが可能になりました。さらに、[静的なエクスポート](https://nextjs.org/docs/app/building-your-application/deploying/static-exports)が可能なため、サーバーをサポートしていない場所（AWS S3バケットなど）にNext.jsをデプロイできるようになりました。

しかし、これは "ただのJavaScript" ではありません。従って、私たちはより本来の設計思想に近いものを目指したいと考えました。

Next.jsが作成されて以来、私たちはMetaのReactコアチームと緊密に協力し、Reactの基本要素の上にフレームワークの機能を構築してきました。私たちのパートナーシップとReactコアチームによる長年の研究開発によって、Next.jsがReactアーキテクチャの最新バージョン、特に[サーバーコンポーネント](https://nextjs.org/docs/getting-started/react-essentials)を含む形で目標達成の機会を得ることができました。


App Routerでは、おなじみの`async`と`await`の構文を使用して[データをフェッチ](https://nextjs.org/docs/app/building-your-application/data-fetching)するため、新たに学ぶべきAPIはありません。デフォルトでは、すべてのコンポーネントはReact Server Componentsであるため、データフェッチングはサーバー上で安全に行われます。例えば、以下のような感じです。

```jsx:app/page.js

export default async function Page() {
  const res = await fetch('https://api.example.com/...');
  // 戻り値はシリアル化され"ません"
  // Date、Map、Setなどを使用できます。
  const data = await res.json();

  return '...';
}
```

これは極めて重要で、"データの取得は開発者次第"という原則が実現されています。これは、開発者がデータを取得し、任意のコンポーネントを組み立てることができることを示しています。さらに、自社のコンポーネントだけでなく、Server Componentsと統合されてサーバー上で完全に動作するように設計された[Twitter埋め込み](https://github.com/vercel-labs/react-tweet)の`react-tweet`のような、Server Componentsのエコシステム内の任意のコンポーネントも対象となります。
<!-- 第6部ここまで -->

```jsx:app.page.js

import { Tweet } from 'react-tweet';

export default async function Page() {
  return <Tweet id="790942692909916160" />;
}

```

ルーターが[React Suspense](https://react.dev/reference/react/Suspense)と連携しているため、コンテンツの一部がロード中でもフォールバックコンテンツをよりスムーズに表示し、必要に応じて段階的にコンテンツを表示することができます。

```jsx:app/page.js

import { Suspense } from 'react';
import { PostFeed, Weather } from './components';

export default function Page() {
  return (
    <section>
      <Suspense fallback={<p>フィードをロード中...</p>}>
        <PostFeed />
      </Suspense>
      <Suspense fallback={<p>天気をロード中...</p>}>
        <Weather />
      </Suspense>
    </section>
  );
}

```

さらに、ルーターはページナビゲーションを[トランジション](https://react.dev/reference/react/useTransition)としてマークし、ルートのトランジションが中断可能になります。

### 自動的なサーバーレンダリングとコード分割

Next.jsを作成した当初は、開発者がWebpack、Babel、およびその他のツールを手動で設定し、Reactアプリケーションを稼働させることが一般的でした。サーバーレンダリングやコード分割などのさらなる最適化を追加する方式は、カスタムソリューションではほとんど取り入れられていませんでした。Next.jsでは他のReactフレームワークと同様に、これらのベストプラクティスを実装し、適用するための抽象化レイヤーを作成しました。

ルートベースのコード分割とは、`pages/`ディレクトリ内の各ファイルがそれぞれ独立したJavaScriptバンドルに分割されるということです。これによりファイルシステムが軽量化され、初回ページロードのパフォーマンスが向上しました。
<!-- 第7部ここまで -->


このコード分割は、Next.jsのサーバーレンダリングアプリケーションとシングルページアプリケーションの両方にとって有益でした。シングルページアプリケーションにとっても有益であった理由は、アプリケーション起動時に単一の大きなJavaScriptバンドルを読み込むことが多いからです。ただし、コンポーネントレベルのコード分割を実装するためには、開発者は`next/dynamic`を使用してコンポーネントを動的にインポートする必要がありました。

```jsx:app/page.tsx

import dynamic from 'next/dynamic';

const DynamicHeader = dynamic(() => import('../components/header'), {
  loading: () => <p>ロード中...</p>,
});

export default function Home() {
  return <DynamicHeader />;
}

```

App Routerを使用すると、Server ComponentsはブラウザのJavaScriptバンドルには含まれません。また、[Client Components](https://nextjs.org/docs/getting-started/react-essentials#client-components)はデフォルトで自動的にコード分割されます（Next.jsのwebpackまたはTurbopackを使用）。さらに、ルーターアーキテクチャ全体がストリーミングとSuspenseを有効にしているため、サーバーからクライアントへのUIの一部を逐次送信することができます。

例えば、条件分岐を利用することでコードパス全体を分割することができます。この例では、ログアウトしたユーザーに対して、ダッシュボードのクライアント側JavaScriptをロードする必要がなくなります。

```jsx:app/layout.tsx

import { getUser } from './auth';
import { Dashboard, Landing } from './components';

export default async function Layout() {
  const isLoggedIn = await getUser();
  return isLoggedIn ? <Dashboard /> : <Landing />;
}

```
:::message
#### 【補足説明】
上記のコードでは、ログイン状態に基づいて表示するコンポーネントを選択しています。ログインしているユーザーには`Dashboard`コンポーネントを、ログインしていないユーザーには`Landing`コンポーネントを表示します。これにより、ログインしていないユーザーのブラウザには、`Dashboard`コンポーネントのJavaScriptコードは送信されません。
:::
<!-- 第8部ここまで -->

## Turbopack（ベータ版）

[Turbopack](https://turbo.build/pack)は、Next.jsでテストと安定化を進めている新しいバンドルです。Turbopackを使用すると、Next.jsアプリケーションの開発速度（`next dev --turbo`を使用）や、本番ビルドの速度（`next build --turbo`を使用）が向上します。

Next.js 13でアルファ版がリリースされて以来、バグを修正し、不足している機能のサポートを追加してきたため、結果としてTurbopackの採用率は着実に上昇しています。私たちは、[Vercel.com](http://vercel.com/)や多数のVercelの顧客が運用する大規模なNext.jsウェブサイトでTurbopackをドッグフーディングし、フィードバックを集め、安定性を向上させてきました。私たちのチームへのバグ報告に協力していただいたコミュニティの皆様には感謝しています。

それから6ヶ月が経ち、現時点で私たちはベータフェーズに進む準備が整いました。

Turbopackは、まだwebpackやNext.jsと完全に同等の機能を備えているわけではありません。私たちは[このissue](https://github.com/vercel/next.js/issues/49174)でその機能のサポートを追跡しています。しかし、ほとんどのユースケースはサポートされているはずです。このベータ版での私たちの目標は、採用率の増加に伴って発生する可能性のある残りのバグに対応し、将来のバージョンの安定性を確保することです。

Turbopackのインクリメンタルエンジンとキャッシングレイヤーを改善するための投資により、ローカルの開発速度だけでなく、近い将来に本番ビルドの速度も向上する予定です。将来のNext.jsのバージョンでは、`next build --turbo` を実行して即座にビルドできるようになりますので、ご期待ください。本日よりNext.js 13.4で`next dev --turbo`を使用し[Turbopack](https://nextjs.org/docs/architecture/turbopack)のベータ版を試すことができます。

## Server Actions（アルファ版）

Reactのエコシステムでは、フォーム、フォームの状態の管理、データのキャッシュと再検証に関するアイデアについて、多くの革新と探求が行われてきました。時間の経過とともに、Reactはこれらのパターンのいくつかについてより多くの意見を持つようになりました。例えば、フォームの状態を管理するために「[非制御コンポーネント](https://react.dev/learn/sharing-state-between-components#controlled-and-uncontrolled-components)」を推奨しています。


<!-- 第9部ここまで -->

現在のエコシステムのソリューションは、再利用可能なクライアントサイドのソリューションか、フレームワークに組み込まれたプリミティブのどちらかでした。しかし、これまでサーバーの変更とデータプリミティブを組み合わせる方法はありませんでした。Reactチームは、[変更に対するファーストパーティーのソリューションを開発してきました。](https://react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023)

:::message
#### 【補足説明】
「これまでサーバーの変更とデータプリミティブを組み合わせる方法はありませんでした。」という部分では、一般的にクライアントサイドのフレームワークでは、サーバー側でのデータ変更と、それをクライアントに反映するためのデータプリミティブ（基本的なデータタイプや構造）を組み合わせて操作することが難しいという問題を指しています。つまり、サーバー側でデータが変更されたとき、その変更を適切にクライアントサイドに反映させるための効率的な方法が存在しなかったのです。Reactチームが開発した新しいソリューションは、この問題を解決するためのものです。
:::


私たちは、実験的な**Next.jsのServer Actions**のサポートを発表できることをうれしく思います。 Server Actionsにより、APIレイヤーを介さずにサーバー上のデータを変更し、関数を直接呼び出すことが可能になります。

:::message
#### 【補足説明】
「APIレイヤーを介さずにサーバー上のデータを変更し、関数を直接呼び出すことが可能になります。」という部分では、Next.jsのServer Actionsの大きな利点を説明しています。従来、クライアントサイドからサーバーサイドのデータを操作するためには、APIを通じてリクエストを送り、レスポンスを待つ必要がありました。しかし、Server Actionsにより、クライアントサイドから直接サーバー上の関数を呼び出し、データを操作することが可能になります。これにより、API通信による遅延を減らし、アプリケーションのパフォーマンスを向上させることが期待できます。また、APIの管理を大幅に簡素化することができるため、開発効率も向上します。
:::

```jsx:app/post/[id]/page.tsx
// app/post/[id]/page.tsx (Server Component)

import kv from './kv';

export default function Page({ params }) {
  async function increment() {
    'use server';
    await kv.incr(`post:id:${params.id}`);
  }

  return (
    <form action={increment}>
      <button type="submit">Like</button>
    </form>
  );
}

```
:::message

#### 【補足説明】
上記のサンプルコードでは、「いいね」機能を実装しています。incrementという非同期関数が定義されており、この関数を呼び出すと、指定された投稿（post:id:${params.id}）の「いいね」数が1増えます。この関数は、サーバー上で動作することを示す'use server'というディレクティブと共に定義されています。そして、この関数は、<form>要素のaction属性で指定されています。そのため、ユーザーが「Like」ボタンをクリックすると、increment関数が実行され、「いいね」数が増えます。
:::

Server Actionsを使用すると、強力なサーバーファーストのデータ変更、クライアントサイドのJavaScriptのコード削減、そして段階的に強化されたフォームが利用できます。

:::message


#### 【補足説明】
##### 強力なサーバーファーストのデータ変更：
  データの変更ロジックをサーバーサイドに集中させることで、より安全かつ効率的にデータを操作することが可能になります。これは、データの整合性を保つために重要です。
##### クライアントサイドのJavaScriptのコード削減：
  APIレイヤーを介さないため、クライアントサイドのJavaScriptのコードが大幅に削減できます。これにより、ページの読み込み速度が向上し、ユーザー体験が改善されます。
##### 段階的に強化されたフォーム：
  フォームは、データの送信やユーザーとのインタラクションによく使われます。Server Actionsを使用すると、このフォームの動作をサーバーサイドで処理し、より効率的で柔軟なフォームの挙動を実現できます。
:::

```jsx:app/dashboard/posts/page.tsx
// app/dashboard/posts/page.tsx (Server Component)

import db from './db';
import { redirect } from 'next/navigation';

async function create(formData: FormData) {
  'use server';
  const post = await db.post.insert({
    title: formData.get('title'),
    content: formData.get('content'),
  });
  redirect(`/blog/${post.slug}`);
}

export default function Page() {
  return (
    <form action={create}>
      <input type="text" name="title" />
      <textarea name="content" />
      <button type="submit">Submit</button>
    </form>
  );
}

```
:::message
#### 【補足説明】
上記のコードでは、createというServer Action関数が定義されています。この関数は、フォームのデータを受け取り、データベースに新しい記事を挿入します。その後、ユーザーを新しい記事のページにリダイレクトします。このフォームは、create関数をaction属性に指定しているため、フォームが送信されるとcreate関数が実行されます。
:::
<!-- 第10ぶっここまで -->

Next.jsのServer Actionsはデータライフサイクルの他の部分、例えばNext.jsのキャッシュ、インクリメンタル静的再生成（ISR）、クライアントルーターと深く統合するように設計されています。
:::message
#### 【補足説明】

具体的には、Server Actionsはデータの作成、更新、削除などの操作を行い、その結果をNext.js Cacheに反映します。この結果、次回のページアクセス時には既に更新されたデータがCacheから取得できるため、データ取得のパフォーマンスが向上します。また、ISRとの統合により、ページの静的生成とデータの更新を同時に効率的に行うことができます。これにより、サイトのスケーラビリティとパフォーマンスが向上します。さらに、クライアントルーターとの統合により、ブラウザ上でのページ遷移をスムーズに行うことが可能になります。
:::

新しいAPIである`revalidatePath`と`revalidateTag`を通じてデータを再検証することで、変更、ページの再レンダリング、リダイレクトが一回のネットワークラウンドトリップで行われ、アップストリームプロバイダが遅い場合でもクライアント上に正しいデータが表示されることが保証されます。

:::message
#### 【補足説明】

"一回のネットワークラウンドトリップ"とは、クライアントからサーバーへのリクエストとそのレスポンスの一連の流れを指します。ここでは、データの変更とその変更に基づくページの再レンダリングやリダイレクトが一回のネットワークラウンドトリップで行われるため、レスポンス速度が向上します。"アップストリームプロバイダが遅い場合でもクライアント上に正しいデータが表示されること"は、Next.jsのISRやキャッシュ機能により、前回取得したデータを利用してページを表示するため、ユーザーは最新のデータを待つことなくページを閲覧できます。

:::

```jsx:app/dashboard/posts/page.tsx
// app/dashboard/posts/page.tsx (Server Component)

import db from './db';
import { revalidateTag } from 'next/cache';

async function update(formData: FormData) {
  'use server';
  await db.post.update({
    title: formData.get('title'),
  });
  revalidateTag('posts');
}

export default async function Page() {
  const res = await fetch('https://...', { next: { tags: ['posts'] } });
  const data = await res.json();
  // ...
}

```
:::message
#### 【補足説明】

上記のコードでは、updateというServer Action関数が定義されています。この関数は、フォームのデータを受け取り、データベースの記事を更新します。その後、revalidateTagを使用してタグpostsに対応するすべてのデータを再検証します。
:::

Server Actionsは組み合わせ可能に設計されており、Reactコミュニティに所属する誰もがServer Actionsを構築し、公開し、エコシステム内で配布することができます。Server Componentsと同様に、クライアントとサーバーの両方で利用可能な組み合わせ可能な基本要素（プリミティブ）の新時代に私たちはわくわくしています。

:::message
#### 【補足説明】

"組み合わせ可能に設計されている"というのは、Server Actionsがモジュール化されており、それぞれのアクションが独立していて互いに組み合わせて使うことが可能であるという意味です。これにより、開発者は自分のニーズに合わせてServer Actionsを組み合わせ、柔軟なソリューションを作り出すことが可能になります。さらに、"Reactコミュニティの誰もがServer Actionsを構築し、公開し、エコシステム内で配布することができる"ということは、開発者は自分自身の作成したServer Actionsを他の開発者と共有し、それを他のプロジェクトで再利用することが可能であるという意味です。これにより、Reactのエコシステムはより豊かで多様なものになり、開発者間での知識共有やコラボレーションが進みます。
:::

さらに、[Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions)は、Next.js 13.4で今日からアルファ版で利用可能です。Server Actionsを使用することを選択すると、Next.jsはReactの実験的リリースチャンネルを使用します。

```jsx
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    serverActions: true,
  },
};

module.exports = nextConfig;

```
:::message
#### 【補足説明】

上記の設定ファイルにより、Server Actionsの新機能を有効にすることができます。`experimental`オプションの中に`serverActions: true`を設定することで、Server Actionsの使用が可能です。
:::
<!-- 第11部ここまで -->


## その他の改善点

- [ドラフトモード](https://nextjs.org/docs/app/building-your-application/configuring/draft-mode)：ヘッドレスCMSからドラフトコンテンツを取得してレンダリングします。ドラフトモードは`pages`と`app`の両方で動作します。既存のPreview Mode APIは強化され、簡素化されましたが、`pages`に対しては依然として機能します。ただし、`app`に対してはPreview Modeは機能しないため、ドラフトモードを使用してください。


## よくある質問

### App Routerの安定性とは何を意味するのでしょうか？
App Routerを今日安定版としてリリースするとは、私たちの開発作業が完了したわけではないということです。ここでの「安定性」は、App Routerの核となる部分が本番環境に対応しており、私たち自身の内部テストと多くのNext.jsの早期採用者による検証を経ていることを示しています。

私たちは、今後も更なる最適化を行っていきたいと考えています。これには、Server Actionsが完全に安定することも含まれています。今日からどこから学び始め、どのようにアプリケーションを構築すべきかをコミュニティに明確に示すために、私たちはApp Routerのコア部分の安定性に向けて努力を続けています。

App RouterはReactの`canary`チャンネル上で構築されており、Server Componentsなどの機能を導入するためのフレームワークがすでに利用可能です。詳細は[こちら](https://react.dev/blog/2023/05/03/react-canaries)。

### App Routerの安定化はNext.js betaドキュメントにどう影響するのでしょうか？
今日から新しいアプリケーションを構築する際には、App Routerの使用を推奨します。App Routerの説明とともに一から書き直されたNext.js betaドキュメンテーションは、現在[安定したNext.jsドキュメンテーション](https://nextjs.org/docs)に置き換えられました。これにより、App RouterとPages Router間で簡単に切り替えることが可能になります。

App Routerの段階的な導入については、[導入ガイド](https://nextjs.org/docs/app/building-your-application/upgrading/app-router-migration)をご覧いただくことをお勧めします。これにより、App Routerの導入方法を学ぶことができます。

### Pages Routerは廃止されるのでしょうか？
いいえ、そうではありません。私たちは引き続き`pages/`の開発をサポートすることを約束しています。これにはバグ修正、改善、セキュリティパッチなどが含まれ、今後も複数のメジャーバージョンで続けられます。開発者が準備が整った時点で、App Routerを段階的に導入できるように

するためです。

`pages/`と`app/`を本番環境で並行して使用することは、サポートされており、推奨されています。App Routerは[ルートごと](https://nextjs.org/docs/app/building-your-application/upgrading/app-router-migration)に導入することができます。

### Server Componentsが「完成」したということは何を意味するのでしょうか？
Next.jsは、Server Componentsを含むReactアーキテクチャを採用するフレームワークの一つです。App Routerによって提供される体験が、他のフレームワークや新しいフレームワークでこのアーキテクチャを使用することを考えるきっかけとなることを期待しています。

ただし、無限スクロールのようなパターンはまだこのエコシステムで定義されていません。現状では、エコシステムが成長し、ライブラリが作成や更新される間、これらのパターンについてはクライアントサイドのソリューションの使用を推奨しています。
<!-- 第13部ここまで -->

## コミュニティ

Next.jsは、2,600人以上の個々の開発者、GoogleやMetaのような業界パートナー、そして私たちVercelのコアチームの共同作業の結果です。[GitHub](https://github.com/vercel/next.js/discussions)の[ディスカッション](https://github.com/vercel/next.js/discussions)、[Reddit](https://www.reddit.com/r/nextjs/)、[Discord](https://nextjs.org/discord)でコミュニティに参加してください。

このリリースは以下の皆様により提供されました：

**Next.jsチーム**: Andrew, Balazs, Jan, Jiachi, Jimmy, JJ, Josh, Sebastian, Shu, Steven, Tim, and Wyatt.
**Turbopackチーム**: Alex, Donny, Justin, Leah, Maia, OJ, Tobias, and Will.
**そして貢献者たち**: @shuding, @huozhi, @wyattfry, @styfle, @sreetamdas, @afonsojramos, @timneutkens, @alexkirsz, @chriswdmr, @jankaifer, @pn-code, @kdy1, @sokra, @kwonoj, @martin-wahlberg, @Kikobeats, @JTaylor0196, @sebmarkbage, @ijjk, @gnoff, @jridgewell, @sagarpreet-xflowpay, @balazsorban44, @cprussin, @ForsakenHarmony, @li-jia-nan, @dciug, @albertothedev, @DuCanhGH, @feedthejim, @patrick91, @padmaia, @sophiebits, @eps1lon, @reconbot, @acdlite, @cjmling, @nabsul, @motopods, @hanneslund, @tunamagur0, @devknoll, @apeltop, @maranomynet, @y-tsubuku, @EndangeredMassa, @ykzts, @AviAvinav, @adilansari, @wyattjoh, @charkour, @delbaoliveira, @agadzik, @Just-Moh-it, @rodrigofeijao, @leerob, @juliusmarminge, @koba04, @Phiction, @jessewarren-aa, @ryo-manba, @Yovach, and @dylanjha.

:::message alert
　　上記の翻訳及び補足説明について、誤りや過不足が見受けられる場合、コメントいただければ幸いです。適宜、確認し修正を行ってまいります。
:::